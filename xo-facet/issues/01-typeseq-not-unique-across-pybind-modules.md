# 01 — typeseq ids are per-pybind-module, but the registries they key are shared

Status: open
Type: bug

`typeseq::id<T>()` allocates from a counter that pybind11 extension modules each
get a PRIVATE copy of, while `FacetRegistry` and `TypeRegistry` are single and
process-wide. So a type has one id inside a python module and a different one
everywhere else, and two unrelated types in two modules share an id.

## Measured

`typerecd::require_next_id()::s_next_id` is a function-local static in a header,
so it has vague linkage and should be merged process-wide. It is — except in
the pybind modules:

```bash
nm -C .build/xo-object2/src/object2/libxo_object2.so | grep s_next_id
#   u xo::reflect::typerecd::require_next_id()::s_next_id     <- GNU unique, merged

nm -C .build/python/xo/object2*.so | grep s_next_id
#   b xo::reflect::typerecd::require_next_id()::s_next_id     <- local BSS, private copy
```

`u` vs `b` is the whole bug. pybind11 builds extension modules with
`-fvisibility=hidden`, which demotes the unique symbol to a local one.

The registries do NOT have this problem — they live in `libxo_facet.so` and stay
`u`:

```bash
nm -C .build/xo-facet/src/facet/libxo_facet.so | grep -E 'FacetRegistry|TypeRegistry' | grep s_instance
```

That combination is the worst of the two: **one shared registry, keyed by ids
that are not unique.**

## Consequences, in increasing severity

Same type, two answers:

```bash
.build/xo-object2/utest/utest.object2 "occupied-slots-appear-in-the-frame" -s | grep -o '"typeseq": [0-9]*'
#   "typeseq": 10       <- from libxo_object2.so

.build/xo-python -c '
import json, xo.facet as f, xo.object2 as o
fcx = f.configure_all(); fw = f.AllocFlywheel.make_default_app(fcx)
x = o.Float.make(fw, 1.5)
print(json.loads(o.flywheel_frame(fw))["strong"]["slots"])'
#   typeseq 0           <- from xo/object2.so, same DFloat
```

1. **Names are lost.** `TypeRegistry::id2name` on a module-local id returns the
   sentinel, so anything reporting a type by name from python shows
   `_%sentinel%_`. This is how the bug was found.

2. **Ids collide across modules.** `DFloat` is 0 inside `xo/object2.so` and
   `DString` is 0 inside `xo/stringtable2.so` — each module's counter starts
   fresh, so the first type registered in each gets the same id.

3. **Rotation can dispatch to the wrong implementation.** `FacetRegistry`
   lookups are keyed on `_typeseq()`. A handle built inside a pybind module
   carries a module-local id into a registry populated with process-wide ones.
   Today that mostly MISSES rather than mis-hits, because the python modules
   largely do not call `register_impl` (`xo-pyobject2/src/pyobject2/pyobject2.cpp`
   has `//SetupObject2::register_facets();`, commented as "Unnecessary:
   guaranteed by HFloat"). Once `.xo-backlog/pyobject2/issues/07` lifts rotation
   into python, a miss becomes a wrong hit.

## Why it has not bitten yet

Everything within a single module agrees with itself, and cross-module rotation
does not happen yet. The frame in `xo-object2/utest/flywheel_frame.test.cpp`
was the first thing to compare an id computed in a pybind module against a name
registered outside one.

## Design — settled 2026-09-14

Both statics are duplicated, not just the counter:

```bash
nm -C .build/python/xo/object2*.so | grep -E "typerecd::recd<.*DFloat>"
#   b ...::id          d ...::s_armed      <- per-module
nm -C .build/xo-object2/src/object2/libxo_object2.so | grep -E "typerecd::recd<.*DFloat>"
#   u ...::id          u ...::s_armed      <- merged
```

So a partial fix is worse than none. Sharing only `s_next_id` would leave the
per-type memo private, `DFloat` would arm once per module and draw TWO ids from
one counter, and the same type would have two ids in one process. Today each
module is at least self-consistent.

The fix therefore has to stop identity depending on symbol merging at all,
rather than repair the merging.

### Shape

**A shared table keyed by type NAME is the source of truth; the per-type static
is demoted to a cache.** A duplicated cache is then harmless -- it caches the
same answer.

1. **`xo-reflectutil` gains a `src/`** -- its first compiled part.
   - `int32_t typeseq_id_for(std::string_view name)`, non-inline.
   - the installable implementation pointer, defined THERE, not in a header.
   - phase (a): a bootstrap `std::vector`, O(n) lookup, no arena, usable during
     dynamic initialization.
   - `typeseq_install_registry(fn)` to swing the pointer.

2. **`recd<T>()` becomes a cache:**

   ```cpp
   template <typename T>
   static typerecd recd() {
       static const int32_t id = typeseq_id_for(xo::reflect::type_name<T>());
       return typerecd(id, xo::reflect::type_name<T>());
   }
   ```

3. **`TypeRegistry` moves from `xo-facet` to `xo-arena`**, and gains a
   `DArenaHashMap` for name -> id beside its existing id -> name vector. The move
   is clean: its includes are `typeseq`, `DArenaVector` and three ppsink
   headers, all at or below arena. `FacetRegistry` includes it and simply
   reaches down.

4. **`xo-arena` acquires appcx machinery** for TypeRegistry setup. Cheap:
   `AppConfig`/`AppContext` live in `xo-subsys`, which arena already depends on.
   `ArenaAppcx` becomes the bottom of the context chain, displacing
   `Indentlog2Appcx` -- about 10 tag-list sites across 7 files, mostly test
   mains plus `xo-interpreter2/src/skrepl/skreplxx.cpp`.
   `FacetConfig::type_registry_capacity()` moves to arena's config, which is a
   better home for an arena-backed structure anyway.

5. **Upgrade at `ArenaAppcx` construction:** copy every (name, id) the bootstrap
   already assigned into the hashmap VERBATIM, then swing the pointer.

### Invariants

1. **Ids are never reassigned.** The upgrade is additive. Measured: a trivial
   program linking libxo_object2 draws 12 ids before `main()` begins --

   ```bash
   # first id drawn inside main() was 12, not 0
   ```

   all of them already cached in per-type statics. Reassignment would silently
   invalidate every one.

2. **The upgrade precedes threads.** Stated as the model, so the pointer needs
   no atomic.

3. **Ids stay dense and sequential**, so `TypeRegistry::_id2name`'s
   `DArenaVector` indexing by `seqno()` keeps working.

4. **The table owns its keys.** `type_name_holder<T>::value` is itself a
   per-module header static, so names live at different addresses in different
   modules. The table copies into its own arena rather than storing a borrowed
   `string_view`.

5. **Internal-linkage types are not name-keyed.** A name containing
   `{anonymous}` draws from the counter WITHOUT being inserted. Two TUs'
   anonymous types share a spelling but are different types:

   ```
   TU a:  namespace { struct Widget { int a; }; }        -> "{anonymous}::Widget"
   TU b:  namespace { struct Widget { double x, y; }; }  -> "{anonymous}::Widget"
   ```

   Measured 2026-09-14, same key. Excluding them preserves today's behaviour,
   which is CORRECT for them -- a type with internal linkage cannot be the same
   type in two modules, so it does not need a global id. 12 test files use the
   pattern with the facet machinery, so this is not hypothetical.

### The trap this design exists to avoid

**The implementation pointer must not be a static in a header.** It would be
duplicated per module under `-fvisibility=hidden` exactly as `s_next_id` is
today: each module would get its own pointer and its own bootstrap vector,
module A would upgrade while module B never did, and the original bug would
return wearing a different hat -- with everything looking correct.

That is the whole reason reflectutil gains a compiled part. The note that
"reflectutil stays header-only by design (no static tables)" is the premise this
bug falsifies: header statics do NOT stay single across pybind modules, so
header-only-with-statics was never delivering what it promised.

### Rejected, with reasons

- **Default-visibility annotations.** Would restore `u`, header-only preserved,
  one line. But correctness stays a linker property: every new pybind module is
  hidden-by-default again and nothing notices -- which is how this went
  unobserved since the first python module. Also needs separate verification on
  the macOS host, where Mach-O has no `u` binding, and interacts with python
  loading modules `RTLD_LOCAL`. Reasonable as a stopgap, not as the fix.
- **`TypeRegistry` allocates, staying in xo-facet.** Right concept, wrong
  direction: `xo-facet` depends on `xo-reflectutil`, so typeseq calling
  TypeRegistry inverts levelization. Moving the registry down is what makes the
  concept available.
- **Hash the name instead of allocating.** No shared table needed, but breaks
  the dense-id property `TypeRegistry` indexes on, and trades a visible
  collision for a silent one.
- **Key identity on `std::type_index`.** Already considered and rejected by the
  author -- `typeseq.hpp:30` records that "built-in typeinfo may return false
  negatives across library boundaries when using clang".

## Done when

- the same type reports the same `typeseq` from C++ and from python
- a frame from python names its types instead of `_%sentinel%_`, asserted in a
  python test -- the level where both a library and a pybind module are loaded
  (e.g. `xo-pyobject2/utest`)
- **ids survive the upgrade**: draw an id, install the hashmap, assert the same
  number comes back. The bootstrap is not observable any other way, and this is
  the invariant whose violation is silent
- an anonymous-namespace type in two TUs of one binary still gets two ids
- the regression check is in the tree, because neither the fix nor a review
  stops a future module hiding something else:

  ```bash
  nm -C .build/python/xo/*.so | grep -E "^.* b .*(require_next_id|typerecd::recd).*"
  ```

  Empty is the passing condition. Note this must keep passing even after the
  fix -- the per-type cache is still duplicated, deliberately; what must not
  reappear is a duplicated SOURCE of ids

## Provenance

Found 2026-09-14 while binding `xo.object2.flywheel_frame()` for a browser
animation of AllocFlywheel state. The frame's `type` field showed
`_%sentinel%_` for every slot.
