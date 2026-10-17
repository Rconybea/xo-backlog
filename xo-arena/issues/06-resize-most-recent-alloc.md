# 06 -- resize the most recent allocation in place

Status: open
Type: feature
Raised: 2026-10-17 (RC), from the LogBufferAdapter / TempFlatSink work in
`.xo-backlog/xo-printjson/issues/01`

Give `DArena` the ability to retroactively change the size of its most
recent allocation -- shrink it, or grow it while it is still last -- in
place, without a free/alloc round trip.

## Motivation (RC)

1. **LogBufferAdapter allocates before it knows how much it will use.**
   Since 2026-10-17 it takes what it needs, at least doubling, at least a
   page, capped by the reserve (`char_expand_to`,
   `xo-indentlog2/src/indentlog2/LogBuffer.cpp`) -- not all of
   `available()`.  So during a nested use (a `tostr()` inside a
   `TempFlatSink`'s print) the parent still holds capacity it is not
   using: everything between its `pptr_` and `epptr_`.  With resize, the
   parent's allocation could shrink to what it has written when a nested
   buffer starts, and regrow once that buffer is gone -- no memory idle
   during the nested call.
2. **Variable-size types built incrementally.**  Allocate a provisional
   instance before the final size is known, then resize it as the
   instance is built -- e.g. `DUniqueString` from characters arriving one
   at a time.  Today `DUniqueString::from_view` knows its size up front
   (`xo-stringtable2/src/stringtable2/DUniqueString.cpp`), so this is a
   capability for builders that do not, not a fix to that path.

## What is there

```bash
cd "$(xo-sdlc --umbrella2)"
grep -n 'value_type alloc\|super_alloc\|sub_alloc\|checkpoint()\|void restore\|bool expand' xo-arena/include/xo/arena/DArena.hpp
grep -n 'last_header_' xo-arena/include/xo/arena/DArena.hpp
grep -n 'store_header_v\|retain_header_v\|store_guard_v' xo-arena/src/arena/DArena.cpp
grep -n 'missing DArena::realloc' xo-indentlog2/src/indentlog2/LogBuffer.cpp
```

- An allocation is laid out `< guard >< header >< mem >< pad >< guard >`;
  header and guards exist only when the arena's config asks for them
  (`store_header_flag_`, guard size).  The header records the size.
- `DArena` does NOT generally remember its last allocation:
  `last_header_` is retained only for a `super_alloc` (so `sub_alloc`s
  can accumulate into it) -- `retain_header_v` is true only for that mode.
- The one resize in the tree is LogBufferAdapter's workaround for "the
  missing DArena::realloc()": `restore(buf_ckp_)` then `alloc()` again,
  checking the result landed where the old buffer was.  It works only
  while the buffer is the arena's last allocation, and it re-runs the
  whole alloc path (header, guard) to change one size.
- Allocation through the `AAllocator` facet (`xo-facet/include/xo/facet/alloc/AAllocator.hpp`)
  is how `DUniqueString` reaches an arena (`mm.super_alloc(..)`), so use
  case 2 needs the capability at the facet as well as on `DArena`.

## Decisions (RC, 2026-10-17)

- **`DArena::realloc_last()`** -- a `DArena` method, NOT exposed through
  `AAllocator`.
- **No new state for "the last allocation".**  `last_header_`, retained for
  a `super_alloc`, already is the last allocation; `realloc_last()` works
  from it.

Consequences, to resolve when building (not decided here):
- `last_header_` is retained only for `super_alloc` (`retain_header_v`), and
  only when `store_header_flag_` is set.  So either `realloc_last()` applies
  to an allocation made with `super_alloc`, or a header-less arena (and a
  plain `alloc`) is out of scope.  LogBufferAdapter today makes its buffer
  with plain `alloc()` -- it would switch to `super_alloc()` to qualify.
- Not on `AAllocator`: use case 2 (`DUniqueString`, which allocates through
  `obj<AAllocator>`) would need to reach the `DArena` directly, or stay
  out of scope for now.

**Alternative (RC, 2026-10-17): retain `last_header_` unconditionally**,
for every alloc mode, not only `super_alloc`.  Then a plain `alloc()`
qualifies too -- LogBufferAdapter would not need to switch to
`super_alloc()`.  And `realloc_last()` could support header-less arenas as
well.  Two requirements hold either way:

- **Consistency:** when the arena stores headers, `realloc_last()` updates
  the header's size, as `super_alloc` / `sub_alloc` already do, so that
  iterating the arena's allocations (header to header) still works.
- **Origin:** `realloc_last()` must know the start of the allocation it is
  resizing -- the origin the new size is measured from.  With headers,
  `last_header_` gives it.  Without headers there is no header to point at,
  so the origin has to come from somewhere else: passed by the caller (the
  address `alloc` returned), or recorded by the arena.  Recording it is
  new state, which the "no new state" decision above avoided -- so
  header-less support and that decision pull against each other.

## Design questions

1. **API shape.**  *(Answered: `DArena::realloc_last()`.)*  `resize_last(mem, new_z) -> bool`, taking the address
   `alloc` returned; or a handle `alloc` returns; or separate
   `shrink_last` / `grow_last`.  Growth beyond committed space would go
   through `expand()` and is bounded by the reserve.
2. **How "last" is established.**  *(Answered: from `last_header_`, no new
   state -- see Decisions.)*  `mem` is last iff its end -- size,
   padding, trailing guard -- is `free_`.  Checkable without new state; or
   track the last allocation explicitly (generalising `last_header_`).  A
   call on anything but the last allocation must refuse, not corrupt.
3. **Header and guard bookkeeping.**  Update the header's size, move the
   trailing guard, keep alignment and `alloc_info()` truthful.  How this
   interacts with `super_alloc` / `sub_alloc` groups, whose single header
   covers several allocations.
4. **The LogBufferAdapter protocol for use case 1.**  Who shrinks the
   parent: the nested adapter (which would need to find the parent), or
   the parent when it hands control away (it cannot tell)?  The parent's
   `epptr_` must follow, so it cannot be done behind its back.  May be
   simplest as an explicit step in `TempFlatSink` / `tostr()` nesting.
5. **Facet exposure.**  *(Answered: not exposed through `AAllocator`.)*  Whether `AAllocator` gains the method, and what
   allocators that cannot do it (non-bump) answer.

**Done when:**
- `DArena` can shrink and grow its most recent allocation in place, with
  and without headers/guards, refusing on any other allocation -- tested
  for each configuration, `alloc_info()` correct afterwards
- LogBufferAdapter's restore-and-realloc workaround is replaced by it
- use case 1 or 2 adopts it, or a decision records why not yet
