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

The weak set has the same defect, for the same reason: `add_weak_ref` also
always `push_back`s, and `remove_weak_ref` also only clears.

All four claims above re-derived 2026-09-12; each still held.

## Shape

Design settled 2026-09-12. **The free lists live in `DHandleStore`**, which
owns the ref vectors — `AllocFlywheel` only forwards. Strong and weak are
treated symmetrically.

1. `DHandleStore` gains a free list per ref vector, each a
   `DArenaVector<handle_index_type>` holding the index positions of cleared
   slots in the corresponding vector.

2. The ctor takes them, rather than building them, matching how it already
   receives `strong` and `weak`:

   ```
   DHandleStore(Storage && storage,
                DArenaVector<Handle> && strong,
                DArenaVector<handle_index_type> && strong_freelist,
                DArenaVector<Handle> && weak,
                DArenaVector<handle_index_type> && weak_freelist)
   ```

3. **Each free list's capacity equals that of the vector it serves.** At most
   every slot is free, so the push in `remove_*_ref` cannot overflow and
   release stays infallible. The capacity is DERIVED, not chosen: `make_app`
   sizes each free list arena at `capacity * sizeof(handle_index_type)` — far
   smaller than the ref arena, an index against a `obj<ATop>` — so no new
   `ArenaConfig` appears in `AllocFlywheel::make_app`'s signature.

4. `add_*_ref` consults the free list before `push_back`: pop an index, assign
   into that slot, return it. `push_back` only when the free list is empty.

5. `clear()` must clear the free lists too. Required, not optional: stale
   indices across a `clear()` have `add_strong_ref` pop an index past the new
   `size()` and assign through `DArenaVector::operator[]`, which is `noexcept`
   and unchecked — silent corruption, surfacing far from its cause.

6. **`remove_*_ref` guards against double release.** Today it checks only
   `ix < size()`; with a free list, calling it twice on one index pushes that
   index twice and two handles later share a slot. The guard needs an
   emptiness predicate on the Handle — `DHandleStore`'s documented contract is
   currently only "Handle provides `.clear()`"
   (`sed -n '14,20p' xo-facet/include/xo/facet/handlestore/DHandleStore.hpp`),
   so `obj<>` gains one. Worth the extra method: slot integrity is exactly
   what the v2 collector will depend on.

7. `AllocFlywheel::remove_strong_ref(ix)` / `remove_weak_ref(ix)` forwarding to
   the store, and `~ObjectHandleBase` calling the strong one.

8. `AllocFlywheel::strong_root_count()` so the harness can assert on pinning.

Keep the free list a separate `DArenaVector<size_type>`. Threading it through
the cleared slots' data words costs no memory, but a collector scanning
`strong_refs_` for roots would then have to distinguish roots from list links —
a trap set for the v2 collector work. (Unchanged by the symmetry decision, and
the reason item 6 is worth its extra method.)

## Consequence: the pool report grows from three to five

`DHandleStore::visit_pools()` walks its vectors, so the free lists appear
there too — `store`, `strong`, `strong-free`, `weak`, `weak-free`, names to be
fixed when written. Two places pin the current three:

```bash
grep -rn 'store.*strong.*weak' xo-pyobject2/utest/test_pyobject2.py
grep -n 'visit_pools' xo-facet/include/xo/facet/handlestore/DHandleStore.hpp
```

`test_reports_the_three_pools_in_order` asserts the list exactly and must be
updated with this change. `xo-pyobject2/example/ex1/ex1.py` prints pool names
but does not pin them, so it needs no edit.

`contains()` enumerates the same vectors as `visit_pools()` and needs the same
addition, or it answers false for an address in a free list:

```bash
grep -n 'bool contains' -A 2 xo-facet/include/xo/facet/handlestore/DHandleStore.hpp
```

Any member that walks `strong_refs_`/`weak_refs_` is a candidate; those two are
the ones present as of 2026-09-12, so re-run that grep rather than trusting
this list.

**Files:**
- Modify: `xo-facet/include/xo/facet/handlestore/DHandleStore.hpp` (free lists,
  5-arg ctor, `clear()`, double-release guard, `visit_pools()`, `contains()`)
- Modify: `xo-facet/include/xo/facet/AllocFlywheel.hpp`, `src/facet/AllocFlywheel.cpp`
  (forwarding, derived free list arenas in `make_app`, `strong_root_count()`)
- Modify: `xo-facet/include/xo/facet/obj.hpp` (emptiness predicate)
- Modify: `xo-facet/src/facet/ObjectHandle.cpp` (`~ObjectHandleBase`)
- Test: `xo-facet/utest/objectmodel.test.cpp` (beside `objecthandle-nontop-facet`)
- Test: `xo-pyobject2/utest/test_pyobject2.py` (pool list; the release assertion
  below)

**Done when:**
- dropping a handle returns its slot, and `strong_root_count()` falls
- the python suite (`xo-pyobject2/utest`, added by `06`) asserts it: `del x`
  followed by a collection lowers the count.  `06` could not include this,
  having nothing to release and nothing to observe
- creating and dropping N handles in a loop, N far above the root set's element
  capacity, does not exhaust it — for the weak set as well as the strong
- releasing the same index twice does not hand one slot to two handles
- `clear()` followed by `add_strong_ref` allocates a valid slot
