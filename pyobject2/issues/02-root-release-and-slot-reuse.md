# 02 — a handle never drops its root, and slots are never reused

Status: open
Type: bug
Milestone: pyobject2

`~ObjectHandleBase` is `= default` (`xo-facet/src/facet/ObjectHandle.cpp`), so a
handle pins its object for the life of the flywheel. Python refcounting then has
no effect on rootedness, which defeats the harness's main purpose: watching an
object become collectable.

`AllocFlywheel` also exposes no way to release — only `add_strong_ref`:

```bash
grep -n 'strong_ref\|weak_ref' xo-facet/include/xo/facet/AllocFlywheel.hpp
```

The store below it already has the release half:

```bash
grep -n 'remove_strong_ref' -A 5 xo-facet/include/xo/facet/handlestore/DHandleStore.hpp
```

but it only nulls the slot. `add_strong_ref` always `push_back`s, and a
`DArenaVector` fixes capacity at construction
(`sed -n '18,24p' xo-arena/include/xo/arena/DArenaVector.hpp`), so a REPL loop
that creates and drops handles exhausts the root set rather than reusing it.

## Shape

1. `AllocFlywheel::remove_strong_ref(ix)` forwarding to the store.
2. `~ObjectHandleBase` calls it.
3. A free list of indices, consulted by `add_strong_ref` before `push_back`.
4. `AllocFlywheel::strong_root_count()` so the harness can assert on pinning.

Keep the free list a separate `DArenaVector<size_type>`. Threading it through
the cleared slots' data words costs no memory, but a collector scanning
`strong_refs_` for roots would then have to distinguish roots from list links —
a trap set for the v2 collector work.

**Files:**
- Modify: `xo-facet/include/xo/facet/AllocFlywheel.hpp`, `src/facet/AllocFlywheel.cpp`
- Modify: `xo-facet/include/xo/facet/handlestore/DHandleStore.hpp`
- Modify: `xo-facet/src/facet/ObjectHandle.cpp`
- Test: `xo-facet/utest/objectmodel.test.cpp` (beside `objecthandle-nontop-facet`)

**Done when:**
- dropping a handle returns its slot, and `strong_root_count()` falls
- the python suite (`xo-pyobject2/utest`, added by `06`) asserts it: `del x`
  followed by a collection lowers the count.  `06` could not include this,
  having nothing to release and nothing to observe
- creating and dropping N handles in a loop, N far above the root set's element
  capacity, does not exhaust it
