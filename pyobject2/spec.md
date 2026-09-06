# pyobject2 — exposing faceted objects to python

Status: open
Type: spec
Milestone: pyobject2

Bind fomo objects (`xo-object2`'s `Float`, `Integer`, `List`, ...) to python, so
a REPL can construct them, hold them across allocation, render them, and watch
what pinning does to the root set.

## Purpose

**A harness, not a product API.** Python drives and inspects the c++ object
model; fidelity to fomo beats feeling pythonic. This decides several things
below — notably that objects stay opaque handles rather than converting to
python values at the boundary, and that observing the root set is a feature
rather than an implementation leak.

## The strategy, and what it replaces

The earlier plan (recorded in the fomo notes) was to extend `genfacet` to emit a
per-facet handle class `H<Foo>`, whose forwarding methods python would then be
bound to. **That layer is unnecessary**, for a reason that is checkable:

```bash
sed -n '75,85p' xo-facet/include/xo/facet/OObject.hpp   # OObject(DataPtr): ISpecific tmp; memcpy
```

`OObject<AFacet,DRepr>` with a concrete `DRepr` materializes its own interface
bytes in the constructor. So in a pybind TU, where `DRepr` is known statically,
`obj<APrintable,DFloat>(p)` is a complete fat pointer built from nothing but the
data pointer — no `FacetRegistry` lookup, no generated forwarder. `genfacet`
would be generating a forwarding layer to sit underneath pybind's `.def()`
lambdas, which are already a forwarding layer.

What repeats instead is *per facet*, so the abstraction goes there: a **binder
template** per facet, adding that facet's methods to any pybind class whose repr
implements it. Cost is O(facets + reprs) rather than O(facets x reprs), and it
needs no generator.

```cpp
template <typename DRepr, typename PyCls>
void bind_printable(PyCls & cls) {
    cls.def("pretty", [](const H<DRepr> & h, PpSink & sink) {
        h.template _native_as<APrintable>().pretty(sink);
    });
}
```

A binder may assume only that `DRepr` implements its own facet — it never names
another — so binders compose in any order, and a new facet is one new file.

## The handle

`DObjectHandle<AFacet,DRepr>` (`xo-facet/include/xo/facet/ObjectHandle.hpp`) is
the proxy. It pins a strong root in an `AllocFlywheel` and recovers the typed
`obj` on demand.

Two properties are load-bearing, both established by
`utest.facet "objecthandle-nontop-facet"`:

1. **Narrowing keeps the concrete vtable.** The flywheel's slot is `obj<ATop>`
   (`grep -n 'HandleStore =' xo-facet/include/xo/facet/AllocFlywheel.hpp`), so a
   handle narrows on the way in via the variant constructor. Every facet
   inherits `ATop` and nothing else — one non-virtual chain, vptr at offset 0 —
   so the interface pointer stored in the slot is still `IFacet_DRepr`, and
   `_typeseq()` / `_drop()` still reach `DRepr` after the facet is forgotten.

2. **Recovery rebuilds from the data pointer, never from the slot's bytes.**
   `obj<ATop,DRepr>` cannot exist — `ITop_Any` specializes only the variant:

   ```bash
   grep -n 'FacetImplementation<ATop' xo-facet/include/xo/facet/top/ITop_Any.hpp
   ```

   so `_native()` constructs `obj<AFacet,DRepr>` from the slot's `void*`. That is
   correct where a reinterpret_cast is not (the slot's interface bytes name
   whichever facet `make_strong_ref` was handed), and it is free, since
   `FacetImplType<AFacet,DRepr>` is resolved at compile time.

   It also makes a rule structural rather than advisory: the data pointer is
   re-read from the slot on every call, so nothing can cache a pointer across an
   allocating call. That matters once a moving collector records relocations in
   the slot.

### Keyed by representation, recovered per facet

A python class covers one representation but several facets, so the handle it
holds cannot be keyed to one facet: `bind_printable` needs
`obj<APrintable,DRepr>` where `bind_sequence` needs `obj<ASequence,DRepr>`.

The handle is therefore keyed on `ATop` -- what the slot actually holds:

```cpp
template <typename DRepr> using H = DObjectHandle<ATop, DRepr>;
```

This is legal even though `obj<ATop,DRepr>` does not exist, because members of a
class template instantiate lazily: `_native()` is simply never called on an
`ATop`-keyed handle. Each binder instead recovers its own facet through a
template member (ticket 08):

```cpp
template <typename AOther> obj<AOther,DRepr> _native_as() const;
```

Verified to compile against `DFloat` + `APrintable` on 2026-09-06.

## Lifetime

A handle's destruction drops its root: `del o` in python makes the object
collectable. That is the property a GC harness most needs, and it is why the
"pin forever, discard the whole flywheel" alternative was rejected.

`DHandleStore::remove_strong_ref(ix)` already exists and nulls the slot; what is
missing is **slot reuse** — `add_strong_ref` always `push_back`s, and a
`DArenaVector` fixes capacity at construction, so a REPL loop exhausts the root
set. A free list of indices is needed. Keep it a separate
`DArenaVector<size_type>` rather than threading it through the cleared slots:
the slot-threaded version costs no memory but would require a future collector
scanning `strong_refs_` to know which entries are list links.

## Modules

| Module | Ships | New? |
|---|---|---|
| `xo-pyfacet` | `bind_top`, handle glue, `AllocFlywheel` class | exists; gains installed headers |
| `xo-pyprintable2` | `bind_printable` | new |
| `xo-pyobject2` | `bind_sequence`, the repr classes | new |

Binders live in the `xo-py*` module mirroring the facet's own subsystem, holding
the 1:1 convention that `xo-pyfacet` / `xo-pyindentlog2` / `xo-pyreactor2`
already follow, and keeping pybind11 out of the c++ subsystems.

Header-installing `xo-py*` modules are an established pattern, not a new one —
`xo-pyutil` ships `xo/pyutil/pyutil.hpp` and five `xo-py*` modules consume it,
`xo-pyfacet` among them:

```bash
find xo-py*/include -type f | grep -v README
grep -rln 'xo/pyutil' --include=*.cpp xo-py*/src
```

(An earlier reading of this said such modules would be a first for the tier. That
was drawn from `xo-pyreflect`, whose `include/` holds only a README, and
generalized without checking the others.)

`xo-pyobject2` must `py::module_::import("xo_pyindentlog2")` at init, since
`bind_printable` gives methods taking `PpSink&`, whose pybind class is registered
there. Precedent: `sed -n '20,30p' xo-pyfacet/src/pyfacet/pyfacet.cpp`.

## Python surface

One class per repr, named without the `D`: `Float`, `Integer`, `Boolean`,
`List`, `Array`, `Dictionary`, `Struct`. Construction is a static factory taking
the flywheel, which is where the allocator view is built:

```cpp
auto alloc = with_facet<AAllocator>::mkobj(&fw->storage());
DFloat::_box(alloc, x);
```

`AllocFlywheel` needs no change for this — `storage()` already returns `DArena&`.

- `ScmRuntimeError`, not `RuntimeError`: the REPL habit is
  `from xo_pyobject2 import *`, and shadowing the builtin with a class that is
  not even raisable is a trap.
- Equality is **identity** (same flywheel, same data pointer). Value equality
  waits on `AEquable`, which is scaffolded with no methods
  (`grep -c 'virtual' xo-equable2/include/xo/equable2/detail/AEquable.hpp`).

## Deferred to v2, and the gate

`collect()`, `bind_gcobject`, and any erased `Obj` type wait until facet rotation
via `FacetRegistry` is lifted into python. Recorded because it decides whether v2
is additive or a rewrite.

Levelization forces part of this anyway: `AllocFlywheel` is below `ACollector`,

```bash
grep -n add_subdirectory CMakeLists.txt | grep -E 'xo-(facet|alloc2)\)'
```

so `fw.collect()` can never be a flywheel method; collection has to be bound from
a module at or above xo-alloc2, taking flywheel and collector separately.

## Not in scope

Generalizing `AllocFlywheel` storage from `DArena` to `obj<AAllocator>`. When it
happens the flywheel will own the allocator polymorphically and store an
`obj<AAllocator>` over it. Nothing here blocks on it: binders and handles never
touch storage, and the factories build their own `obj<AAllocator>` view.
