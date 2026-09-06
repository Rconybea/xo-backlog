# 05 — xo-pyobject2: one python class per representation

Status: open
Type: feature
Milestone: pyobject2
Depends: 03 (installed binder headers)

The module the rest of this milestone exists for. One pybind class per
`xo-object2` representation, each assembled from binders:

```cpp
auto c = py::class_<H<DFloat>>(m, "Float");   // H<DRepr> = DObjectHandle<ATop,DRepr>
bind_top<DFloat>(c);
bind_printable<DFloat>(c);
```

Reprs to cover:

```bash
ls xo-object2/include/xo/object2/D*.hpp
```

Naming: drop the `D` (`DFloat` -> `Float`), except `DRuntimeError` ->
**`ScmRuntimeError`**. The REPL habit is `from xo_pyobject2 import *`, and
shadowing the builtin `RuntimeError` with a class that cannot even be raised is
a trap worth one ugly name.

## Construction

A static factory per class, taking the flywheel and building the allocator view
from its arena:

```cpp
.def_static("make", [](bp<AllocFlywheel> fw, double x) {
    auto alloc = with_facet<AAllocator>::mkobj(&fw->storage());
    return H<DFloat>::make_strong_ref(fw, with_facet<APrintable>::mkobj(DFloat::_box(alloc, x)));
})
```

`AllocFlywheel` needs no change: `storage()` already returns `DArena&`, and
`IAllocator_DArena` comes in from xo-alloc2, which this module already depends on
through xo-object2 (`xo-deps --why=xo-object2:xo-alloc2`).

`SetupObject2::register_facets()` runs at module init, and the module imports
`xo_pyindentlog2` so `PpSink&` signatures resolve.

## Equality

Identity — same flywheel, same data pointer. Value equality waits on `AEquable`,
which is scaffolded with no methods:

```bash
grep -c virtual xo-equable2/include/xo/equable2/detail/AEquable.hpp
```

**Done when:**
- `Float.make(fw, 3.5)` renders as `3.5` through a `PrettySink` from python
- every repr above has a class, and each class's facets are one binder line each
