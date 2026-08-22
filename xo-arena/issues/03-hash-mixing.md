# 03 — `DArenaHashMap` uses `std::hash` output raw; sequential int keys degrade `find()` to O(N)

Status: diagnosed 2026-08-22
Type: performance bug

`std::hash<T>` is not required to mix. In libstdc++ `std::hash<int>` is the
identity. `DArenaHashMap` splits the hash as

```
h1 = h >> 7      // selects the probe's starting group
h2 = h & 0x7f    // stored in the control byte
```

and applies no mixing of its own — `hash_(key)` is used raw at every call site
(`xo-arena/include/xo/arena/DArenaHashMap.hpp:144, 305, 533, 563, 708, 972`).

With identity hashing, `h1 = key >> 7`, so **128 consecutive keys share a home
slot** and keys `0..N` occupy only `N/128` distinct homes. At 16000 sequential
keys in a 32768-slot table every key homes into slots `0..125`; the table is 49%
loaded but the occupied region is one enormous run, and every probe walks it.

## Measured 2026-08-22

Same table, same keys, same build — the only difference is a splitmix64
finalizer applied downstream of `std::hash`:

| live entries | capacity | `find()` with `std::hash<int>` | with mixing | ratio |
|---|---|---|---|---|
| 250 | 512 | 147.6 ns | 12.2 ns | 12x |
| 1000 | 2048 | 516.1 ns | 11.5 ns | 45x |
| 4000 | 8192 | 2048.1 ns | 12.5 ns | 164x |
| 16000 | 32768 | 7983.4 ns | 13.3 ns | 602x |

The mixed column is flat — `find()` is O(1), as designed. The raw column is
linear in table size: this is a hash-quality defect, **not** a defect in `find()`
or in the probe loop. That distinction is the whole point of the second column;
without it the obvious (wrong) conclusion from the first column alone is that
`find()` is O(N).

These figures are from a standalone **-O2** build. The in-tree build is
`CMAKE_BUILD_TYPE=debug` (`-g`, no `-O`), where the same program reports 8.4x /
31.9x / 116x / 448x — the shape is identical and the magnitude is compressed,
because the unoptimised mixed baseline is ~16 ns rather than ~12 ns. **Compare
shapes, not absolute numbers**, unless the build type is stated.

The mixing used, for reproduction:

```cpp
struct MixHash {
    std::size_t operator()(int x) const noexcept {
        std::uint64_t z = std::hash<int>{}(x) + 0x9E3779B97F4A7C15ull;
        z = (z ^ (z >> 30)) * 0xBF58476D1CE4E5B9ull;
        z = (z ^ (z >> 27)) * 0x94D049BB133111EBull;
        return (std::size_t)(z ^ (z >> 31));
    }
};
```

This is now a checked-in program, `xo-arena/bench/hash_mixing.cpp`:

```bash
xo-build -q --configure --build --with-examples xo-arena
./xo-arena/.build/bench/arena_bench_hash_mixing
```

See `xo-arena/bench/README.md` for the standalone -O2 invocation that produced
the table above.

## Sequential integer keys are the normal case, not a corner

`utest/` uses `0..n-1` throughout, and ids, indices, offsets and timestamps are
exactly what an arena-backed map gets used for. The measured tests pass because
they check correctness, and correctness is unaffected.

## Interaction with tombstones — unverified in C++

In the JS model behind `xo-arena/docs/_static/hashmap-lifecycle.html`, clustering
also drives tombstone creation: with a mixed hash ~32% of erases needed a
tombstone, with the identity hash ~99%, because the rule is "is the run through
this slot 16 or longer". Plausible, and it follows from the same clustering, but
**it has not been measured against the C++ implementation** — the model is a
reimplementation, not the code. See issue 02.

## Design questions to settle before implementing

- **Where.** Mixing inside `DArenaHashMap` (one private `_mix()` wrapping every
  `hash_()` call) keeps `std::hash` usable as the default. Mixing in a supplied
  functor pushes the burden onto callers and will be forgotten.
- **Cost to good hashes.** A caller who already supplies a well-mixed hash pays
  for the finalizer twice. abseil mixes unconditionally and accepts this;
  splitmix64's finalizer is ~1 ns against the ~12 ns floor above.
- **Seeding.** abseil also mixes in a per-table seed, which defends against
  HashDoS and stops pathological key sets being reproducible across runs. That
  is a separate decision from mixing at all, and it makes iteration order
  non-deterministic between runs — check nothing in xo depends on that before
  adopting it.
- **`h2` quality.** Mixing fixes `h1` spread; confirm it also spreads `h2`,
  since a degenerate `h2` costs a false-positive comparison per candidate rather
  than a longer probe.

## Files

- `xo-arena/include/xo/arena/DArenaHashMap.hpp:144, 305, 533, 563, 708, 972` —
  the raw `hash_()` call sites
- the `h1`/`h2` split is written inline at each call site (`h >> 7`,
  `h & 0x7f`), not factored into a helper — six places to keep consistent if the
  split ever changes. `DArenaHashMapUtil.hpp` holds the control-byte constants
  (`c_group_size`, `c_control_stub`, `c_empty_slot`, `c_tombstone`) but not the
  hash split.

## Done when

`find()` cost is flat in table size for sequential `int` keys with the default
hash — i.e. the table above collapses to one column. A regression test that
fails on the unmixed version would be worth more than the fix.
