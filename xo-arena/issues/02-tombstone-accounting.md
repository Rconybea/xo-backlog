# 02 — `erase()` leaves tombstones unaccounted; no rehash-in-place

Status: diagnosed 2026-08-22
Type: performance

`DArenaHashMap::erase()` (`xo-arena/include/xo/arena/DArenaHashMap.hpp:567`)
carries

```cpp
// TODO: maybe want to count tombstones, drive table
//       growth
```

Nothing counts them, and `load_factor()` is live entries only
(`xo-arena/include/xo/arena/hashmap/HashMapStore.hpp:86`):

```cpp
float load_factor() const noexcept { return size_ / static_cast<float>(n_slot_); }
```

abseil, by contrast, charges tombstones against the growth budget and has
`DropDeletesWithoutResize` — a rehash-in-place that clears tombstones without
doubling capacity. xo has neither.

## The reading that turned out to be wrong

The obvious story is: *tombstones accumulate without bound, so a probe
eventually finds no empty slot at all, and because `load_factor()` ignores them
nothing triggers a growth to clean up.* That story is wrong, and it is worth
recording because it is the first thing anyone will reason their way to (and I
did).

It is wrong because **inserts reclaim tombstones**. `_try_insert_aux` remembers
the first tombstone on the probe path and lands there instead of on the
terminating empty, so in a steady-state workload erases mint tombstones and
inserts retire them at the same rate. The population reaches equilibrium rather
than growing.

Measured — constant live set, keys churning, so `size()` never rises and no
growth is justified by live data:

```bash
# see the source block at the end of this ticket
$ ./tomb-sat 100 200000
after fill:   size 100  capacity 128  load 0.781
final:        size 100  capacity 128  load 0.781   growths 0
$ ./tomb-sat 1000 200000
final:        size 1000  capacity 2048  load 0.488   growths 0
```

200k churn operations, zero growths, at two sizes. Repeated with a mixing hash
at 400k operations: also zero growths. **Saturation was not reached and is not
demonstrated.**

## What is actually true

Tombstones impose a steady-state *lookup* cost, because a probe must scan past
them. Two tables holding the same 4000 entries at the same capacity, one filled
once and one churned through 400k erase/insert cycles:

```
mixed hash: live 4000 churn 400000 | cap after fill 8192 -> final 8192 (growths 0)
            find churned 16.82 ns   fresh 12.64 ns   ratio 1.330x
mixed hash: live 1000 churn 400000 | cap after fill 2048 -> final 2048 (growths 0)
            find churned 14.99 ns   fresh 11.83 ns   ratio 1.268x
```

So: **a bounded lookup penalty that does not compound**, not a correctness or
capacity hazard. The figures above are a standalone -O2 build; the in-tree
debug build reports 1.22x / 1.14x for the same runs. Compare shapes, not
absolute numbers, unless the build type is stated.

**Measure this with a mixing hash, not the default.** With `std::hash<int>` and
sequential keys the same comparison reads 1.07x — but only because both tables
are already 40x–600x slower from clustering (issue 03), which swamps the effect
being measured. A number taken in that regime is meaningless.

## Unverified

- Whether an adversarial workload *can* drive tombstones to fill the table.
  Equilibrium was observed for random-victim churn with fresh keys; it is not a
  proof. A workload whose inserts systematically probe different paths than its
  erases would be the thing to try.
- Whether the ~30% drag matters for any real xo consumer. No profile has
  attributed anything to it.

## Files

- `xo-arena/include/xo/arena/DArenaHashMap.hpp:567` — the TODO, in `erase()`
- `xo-arena/include/xo/arena/hashmap/HashMapStore.hpp:86` — `load_factor()`
- `_try_insert_aux` — the tombstone-reuse path that produces the equilibrium

## Done when

Either:

- a tombstone counter exists, growth is driven by `size_ + tombstones_`, and a
  rehash-in-place path clears tombstones when they dominate but live count is
  low; **and** the churned-vs-fresh ratio above measurably improves; or
- a note in the header records that the drag is bounded and deliberately
  accepted, and this ticket closes as won't-fix.

Given the measurement, the second is a legitimate outcome. Do not do the work
because abseil does it — abseil's growth budget also feeds `reserve()` and
`rehash()` semantics that xo does not offer.

## Reproduction

Both programs are checked in as `xo-arena/bench/tombstone_churn.cpp` (and
`hash_mixing.cpp` for issue 03):

```bash
xo-build -q --configure --build --with-examples xo-arena
./xo-arena/.build/bench/arena_bench_tombstone_churn
./xo-arena/.build/bench/arena_bench_tombstone_churn --std-hash   # reproduces the trap
```

`tombstone_churn` fills to a target live-set size, then repeats { erase a random
live key; insert a brand-new key }, reporting any capacity change, and finally
times `find()` over a churned and a never-churned table holding the same keys.
Public API only (`size()`, `capacity()`, `load_factor()`, `find()`, `insert()`,
`erase()`), so no friend access to `HashMapStore` is needed. See
`xo-arena/bench/README.md`.
