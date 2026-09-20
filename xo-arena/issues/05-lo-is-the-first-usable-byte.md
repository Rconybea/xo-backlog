# 05 — nine bugs, one assumption: `lo_` is the first usable byte

Status: done 2026-09-20
Type: bug

Giving `DArena` a back pointer at the base of its storage broke nine places
across six subsystems. They looked like nine bugs. They were one: every piece
of code written before the preamble existed assumed **`lo_` is where usable
storage begins**, and that had been true since `DArena` was written.

Worth recording as one finding because the individual fixes teach nothing --
"add `preamble_z()`" nine times -- while the shape teaches several things, and
the shape is what will recur the next time anything is reserved at the base.

## What changed underneath

`establish_meta_pointer()` writes a `DArena*` at `free_` and advances it;
`establish_initial_guard()` follows. So an arena's layout became

```
[ back ptr 8 ][ initial guard guard_z_ ][ first allocation ... ]
^lo_                                    ^lo_ + preamble_z()
```

`lo_` had meant two things at once -- *start of the mapping* and *start of
usable storage*. The preamble split them, and every site that had conflated
them broke.

## The nine, grouped by HOW they were wrong

Not all the same fix, which is why they were found one at a time rather than
by one grep.

### A. Address arithmetic anchored at `lo_` (3)

| | |
|---|---|
| `DArenaVector::_address_of` / `data()` | element 0 WAS the back pointer. The element write destroyed it, then a move's `fixup_meta_pointer` wrote it back and destroyed element 0 |
| `DArena::begin_header()` | added only `guard_z_`, so iteration began on the back pointer and read it as an `AllocHeader` |
| `arena_streambuf` | `pbase = arena_->lo_`. Dead code -- no caller, no test -- and deleted rather than fixed |

### B. Size arithmetic that omitted the preamble (3)

| | |
|---|---|
| `HashMapStore` | sized its two arenas at bare payload, 8 bytes short |
| `LogBufferAdapter` | `committed_span()` was preamble-relative while `expand_to()`'s argument was `lo_`-relative, so `expand_to(size()+1)` asked for less than was already committed and silently did nothing |
| `DArenaVector::_memory_z` | omitted the trailing guard and the alignment padding as well as the preamble |

### C. The preamble is established LAZILY, so earlier pointers go stale (3)

This is the group that matters, and the one a `committed_z_ > 0` guard does not
always solve.

The preamble is written by the first `expand()`, not by `map()`. Until then
`free_ == lo_` and there is no preamble.

| | |
|---|---|
| `DArena::clear()` | wrote the back pointer unconditionally; on a never-committed arena that address is `PROT_NONE` |
| `DArena::begin_header()` (second bug, same function) | on an uncommitted arena returned `lo_ + preamble_z()` while `end_header()` returned `free_ == lo_` -- **begin 8 bytes past end**, and the walk ran off into unmapped memory |
| `GCObjectStore::snap_move_checkpoint` | captured `free_` per generation. If the space had not committed, it captured `lo_` -- **the address the preamble was then written at.** The forwarding walk started on the back pointer and dispatched on a garbage typeseq |

The first two are fixed with `if (committed_z_ > 0)`, now inside
`establish_meta_pointer()` and `begin_header()`.

**The third cannot be.** The captured pointer was valid when taken and was
invalidated afterwards -- staleness across time, not a read of an uncommitted
address. Fixed by seeding the checkpoint at the first address an object COULD
occupy:

```cpp
gray_lo_v[g] = std::max(to_sp->free_, to_sp->lo_ + to_sp->preamble_z());
```

correct in both states, because `preamble_z()` is a pure function of config and
does not require the preamble to have been written.

## A fourth category: ~20 test assertions

Spread over `xo-arena`, `xo-alloc2`, `xo-gc`, `xo-object2`. All variations on
`allocated() == 0` or `available() == committed()`.

Correct to fail -- an empty arena now costs its preamble -- but two traps in
updating them:

1. **`allocated() == 0` means two different things.** Before first commit it is
   right (nothing consumed); after, it should be `preamble_z()`. A blanket
   replace broke the pre-commit sites. They have to be separated by
   measurement, not by pattern.
2. **`preamble_z()` already INCLUDES the initial guard.** A layout formula that
   opened with `cfg.header_.guard_z_` double-counts it once `preamble_z()` is
   added.

Where a site is reached in both states, the expectation has to say so:

```cpp
REQUIRE(a->allocated() == (a->committed() > 0 ? a->preamble_z() : 0));
```

All of them are now written through `preamble_z()` / `overhead_z()` /
`per_alloc_overhead_z()` rather than literals, so the next change to the
preamble moves them.

## What actually found these

Not review. Each one surfaced as a crash or a failing assertion, usually far
from its cause, and the order was dictated by what the previous fix unblocked.

Two were found only because something ELSE turned their code path on:

- `visit_pools` walks allocation headers only when `store_header_flag_` is set.
  Flywheel storage had it off until `DHandleStore` began requiring it, so the
  `begin_header()` bug went live months after it was written.
- `arena_streambuf` had the bug, no caller and no test. It compiled, so it
  looked maintained.

**A latent bug plus an unrelated change that reaches it is indistinguishable
from a new bug.** Both above read as regressions of the change that exposed
them.

## What would have helped

- **Name the coordinate system.** `LogBufferAdapter`'s methods are now
  `char_expand_to` etc., so a caller mixing usable-bytes with from-`lo_` is a
  naming mismatch at the call site instead of a silent off-by-preamble. The
  cheapest durable defence found here.
- **One accessor, not a formula.** `preamble_z()`, `per_alloc_overhead_z()`,
  `overhead_z()` exist now. Before them, each site did its own arithmetic and
  each got it wrong differently.
- **Establish invariants where they are defined.** The `committed_z_ > 0` guard
  now lives inside `establish_meta_pointer()` rather than being restated by
  each of its three callers -- `clear()` was the caller that forgot.

## The same shape, in the other direction

While fixing these, `map_aligned_range`'s `align_z` turned out to mean THREE
things -- base alignment, extent granularity, and (via `arena_align_z_`) commit
granularity. Feeding it a 2GB base alignment made a 64-byte allocation try to
commit 2GB. See `.xo-backlog/xo-arena/issues/04`.

So the lesson generalises past `lo_`: **a name carrying two meanings is a latent
bug, and stays invisible until something makes the meanings differ.** `lo_`
carried two for as long as they coincided.

## Detection

No live instances remain outside `xo-arena`:

```bash
grep -rn "\.lo_\|->lo_" --include=*.cpp --include=*.hpp xo-*/ | grep -v "^xo-arena/"
```

Surviving hits are legacy `xo-alloc` (its own unrelated `lo_`), `xo-distribution`'s
`Uniform`, and comments. `DArena::_mem_lo()` and private members now make the
raw field harder to reach by accident.
