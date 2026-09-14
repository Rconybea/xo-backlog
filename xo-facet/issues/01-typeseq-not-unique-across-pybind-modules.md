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

## Directions, none settled

1. **Export the symbol from the modules.** Annotate `typerecd::require_next_id`
   (or the whole `typeseq` machinery) with
   `__attribute__((visibility("default")))` so `-fvisibility=hidden` does not
   demote it. Smallest change; needs checking that it survives on macOS too,
   where the fleet has a second host (`.xo-backlog/` host topology note).
2. **Move id allocation behind a compiled boundary.** Give `typeseq` a
   non-inline `require_next_id()` defined in a library, so there is nothing for
   a module to copy. Costs `xo-reflectutil`'s header-only property, which is
   deliberate — see the reflectutil-vs-reflect note.
3. **Allocate ids through `TypeRegistry`**, which is already shared and already
   the thing that maps id to name. Largest change, and arguably where this
   belongs: the counter and the registry are the same concern.

(3) looks most correct, (1) is the one-line stopgap. Do not pick by size alone:
whichever is chosen has to hold for every pybind module, so the check below
belongs in the tree either way.

## Done when

- `nm -C` shows no local copy of the counter in any `.build/python/xo/*.so`
- the same type reports the same `typeseq` from C++ and from python
- a frame from python names its types instead of `_%sentinel%_`
- a test asserts it, at a level where both a library and a pybind module are
  loaded -- which is a python test, e.g. in `xo-pyobject2/utest`

## Provenance

Found 2026-09-14 while binding `xo.object2.flywheel_frame()` for a browser
animation of AllocFlywheel state. The frame's `type` field showed
`_%sentinel%_` for every slot.
