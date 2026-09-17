# 04 — reserve an arena's base on a 2^k boundary, so a pointer can find it

Status: done 2026-09-17
Type: feature

`DArena` can now be asked to place its base address on a power-of-two boundary,
so a caller holding a pointer into the arena recovers the base by masking:

```cpp
base = (uintptr_t)p & ~(align_z - 1);
```

Two independent knobs on `ArenaConfig`:

| | |
|---|---|
| `with_base_align_z(z)` | place the base on a `z`-byte boundary; `z` a power of two, `0` = today's page/hugepage behaviour |
| `with_exclusive_block_flag(b)` | reserve the WHOLE block rather than just `size_` |

They buy different things, which is why they are separate:

- **Alignment alone** recovers the base of a pointer ALREADY KNOWN to belong to
  this arena. It does not let you classify an anonymous pointer -- the rest of
  the block is ordinary free address space, and an unrelated `mmap` landing
  there masks to this arena's base.
- **Exclusivity** closes that: nothing else can be mapped in the block, so
  masking is sound for a pointer of unknown provenance.

Choosing one `k` across every participating arena is an orchestration concern,
deliberately outside `DArena` -- all an arena can do is be capable of taking
part. `DArena` validates only what it can see: the alignment is a power of two,
and `size_ <= base_align_z_` (otherwise the arena spans two blocks and a pointer
in the second masks to the wrong base). Both throw, which is `map()`'s existing
policy; the store-an-`AllocError`-and-return policy belongs to the alloc path,
and `map()` has no vocabulary for a failed `DArena` to return.

## Most of it already existed

`mmap_util::map_aligned_range()` already over-requested by the alignment,
computed the aligned base, and `munmap`ped both the unaligned prefix and suffix.
The feature is a config field, a `max()`, and one parameter split.

## Three conflations, all the same shape

Each was one variable carrying two meanings. They are worth listing together
because the third was found only by the sweep.

**1. base alignment vs extent granularity** (`map_aligned_range`). `align_z` was
both where the base goes and what the extent is rounded up to:

```cpp
size_t target_z = padding::with_padding(req_z, align_z);
```

Identical when the alignment is a page or hugepage -- which is why it went
unnoticed -- but a 1MB request at 2GB alignment would reserve 2GB. Split into
`base_align_z` and `page_align_z`; existing callers pass the same value twice.

**2. base alignment vs commit granularity** (`DArena::expand`). `arena_align_z_`
is a THIRD meaning: the unit `expand()` commits in.

```cpp
std::size_t aligned_target_z = padding::with_padding(target_z, arena_align_z_);
```

Feeding it the base alignment made a 64-byte alloc try to commit 2GB. That
failed loudly for a 1MB arena and, worse, SUCCEEDED for an exclusive one by
committing the whole block. `DArena::map` now passes `page_align_z` there; a
caller wanting the mask reads `config().base_align_z_`.

**3. "has an alignment" vs "asked for one".** The `size_ <= base_align_z`
validation ran unconditionally. With `base_align_z_` defaulting to 0 the
fallback is the page size, so EVERY arena larger than a page threw.

That one is a process failure rather than a coding one. After changing
`DArena::map` only `utest.arena` was rebuilt and re-run; the umbrella build was
green and the feature's own tests passed. `xo-build --sweep` then failed 24
subsystems. Narrowing the re-run to the thing just edited is exactly what the
two-build-paths note warns about, and the guard against it is cheap: run the
sweep, not the target.

## Measured: reserving address space costs no page-table entries

The premise that motivated trimming the over-request was page-table pressure.
It is not that:

| | VmSize | VmPTE | VMAs |
|---|---|---|---|
| baseline | 2,708 kB | 44 kB | 23 |
| after `mmap` 2GB `PROT_NONE` | 2,099,860 kB | **44 kB** | 24 |
| after touching 4 kB | — | **52 kB** | 25 |

PTEs appear on first touch, not on reservation. A reservation costs one VMA and
some address space. So the trim is worth doing -- address space is finite at
47-bit user VA -- but not for the reason assumed, and exclusivity is cheaper
than it looks: 2GB reserved, zero resident, zero page tables.

Recorded in `with_exclusive_block_flag`'s doc comment so the number does not
have to be re-derived.

## Where this is exercised

```bash
.build/xo-arena/utest/utest.arena "[base_align]"
```

Three cases: a 1MB arena at 2GB alignment masks correctly and reserves 1MB; the
exclusive variant reserves the full 2GB; and the two invalid configurations are
refused. All three behaviours falsified -- reverting any one breaks a case.

## Not done

- No consumer yet. The first will presumably be a collector wanting O(1)
  per-arena metadata from an interior pointer, which today costs a linear scan
  over arenas via `DArena::contains()`.
- Classifying an anonymous pointer needs more than masking even with
  exclusivity: something recognisable at offset 0 of the block, so a masked
  candidate can be confirmed. That is the orchestrator's design, not the
  arena's.
- `AllocError` already has `alloc_info_disabled`, but `alloc_info()` on a
  headerless arena segfaults rather than reaching it. Unrelated to this ticket,
  found alongside it, and probably a better answer than the `DHandleStore`
  requirement added on 2026-09-15.
