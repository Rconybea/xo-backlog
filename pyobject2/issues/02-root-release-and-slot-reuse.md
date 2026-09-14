# 02 — a handle never drops its root, and slots are never reused

Status: done
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


## Outcome (2026-09-13)

Implemented as designed, with two corrections to the design's premises and one
addition it did not anticipate.

### Correction: `remove_*_ref` did not compile

The ticket read the release half as present-but-incomplete ("it only nulls the
slot"). That is true of the source text and false of the code: `obj<>` has no
`clear()`, only `reset()`. Both `remove_strong_ref` and `remove_weak_ref` are
non-template members of a class template, so they were never instantiated --
nothing called them -- and the mismatch sat there unreported.

```bash
grep -rn 'remove_strong_ref' --include=*.cpp --include=*.hpp xo-*/   # only DHandleStore, before this change
```

So item 6's premise was doubly wrong: the emptiness predicate it proposed to
ADD already existed (`OObject::operator bool`, data_ != nullptr), and the
`.clear()` it assumed present did not. Net effect on `obj<>`: **no change at
all**. `DHandleStore` now calls `reset()`, and its documented contract says
`reset()` + contextual-bool rather than `clear()`.

Worth generalising: a method on a class template that nothing calls is not
tested, not compiled, and not evidence of anything. The doc comment describing
its requirements was the only thing anyone had read.

### Addition: the handle is move-only

Not in the design, and necessary. `~ObjectHandleBase` now releases
`object_ix_`, so a COPY releases the same index twice. The second release can
land after the slot has been reused, handing one slot to two live handles --
and `DHandleStore`'s guard cannot see it, because by then the slot is
legitimately occupied.

The per-index guard (item 6) and move-only ownership answer different halves:
the guard catches a repeated release of a slot still free, move-only prevents a
second releaser existing. Both are needed; neither substitutes.

pybind11 needs only the move, for by-value returns from the `make` factories.

### Free list sizing is derived from capacity(), not from the config

Item 3 said the capacity is derived from the ArenaConfig. It has to be derived
from the constructed vector's `capacity()` instead: an arena rounds its
reservation up to a page, so `DArenaVector::capacity()` (`reserved()/sizeof(T)`)
exceeds what the requested size implies. Sizing off the request leaves the free
list short by that rounding, and release stops being infallible.

### Falsified, each guard separately

Removing any one of these breaks a test, and restoring it makes them green
again:

| removed | result |
|---|---|
| the emptiness guard in `_remove_ref` | `double-release-does-not-share-a-slot` fails |
| the release in `~ObjectHandleBase` | 3 of 4 C++ cases fail; the python suite SEGFAULTS |
| disarming the source in the move ctor | `objecthandle-move-does-not-double-release` fails |

The python segfault is the ticket's bug reproduced exactly: the 20000-iteration
loop exhausts the root set, `add_strong_ref` returns a null slot pointer, and
`_native()` dereferences it.

Method note, learned the hard way here: **do not use `git checkout -- <file>`
to undo a falsification.** It restores from HEAD, which discards the work in
progress; two of the three falsifications above were first run against a stale
binary and proved nothing. Copy the file aside first.

### The weak set was retired instead of being made symmetric

The design said "strong and weak are treated symmetrically", and it was
implemented that way -- then the weak half was deleted outright (`802ddce5`),
along with `weak_refs_` itself, not merely its free list.

That is the better answer, and the symmetry is what exposed it: writing
`add_weak_ref`/`remove_weak_ref`/`weak_root_count` and a second free list made
it plain that nothing anywhere called any of them. The weak set was dead code
with a promise attached ("sent to a well-defined sentinel state whenever
storage_ is reclaimed"), and nothing implemented the promise either.

So `AllocFlywheel::make_app` now takes two ArenaConfigs, not three.

Worth generalising alongside the `remove_*_ref` finding above, because it is the
same shape seen from the other side: **both halves of this ticket's surface
turned out to be uninstantiated code that read as working.** One did not
compile; the other compiled and was unreachable. A doc comment was the only
evidence for either.

### Pool report: three pools, different ones

`["store", "strong", "strong-free"]` -- the count is unchanged from before this
ticket, but `weak` has been replaced by `strong-free`. The free list takes its
name from the set it serves, so a caller that named its root set gets a matching
name without being told the rule.

`test_reports_the_three_pools_in_order` updated in place. Three places carried
prose that had to follow the same path twice -- to five, then back to three:
`AllocFlywheel::visit_pools`'s doc comment, pyfacet's `visit_pools` docstring,
and `xo-pyobject2/example/ex1/ex1.py`'s comment. None is pinned by a test, which
is exactly why all three went stale; re-read them if the pool set moves again.

### Not fixed here: exhaustion is still UB

Holding more live handles than the root set has slots silently truncates and
then segfaults on read. Measured pre-existing (identical crash with this
change stashed), so out of scope: `.xo-backlog/pyobject2/issues/09`.

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
