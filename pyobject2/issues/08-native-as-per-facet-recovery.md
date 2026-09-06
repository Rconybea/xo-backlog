# 08 — DObjectHandle needs per-facet recovery, not one fixed facet

Status: open
Type: feature
Milestone: pyobject2

`DObjectHandle<AFacet,DRepr>::_native()` returns `obj<AFacet,DRepr>` for the one
facet the handle was keyed to. A python class covers one *representation* but
several facets — `bind_printable` wants `obj<APrintable,DRepr>` where
`bind_sequence` wants `obj<ASequence,DRepr>` — so no single key works.

Keying the handle on `ATop` is the honest choice, since `obj<ATop>` is what the
flywheel slot holds:

```cpp
template <typename DRepr> using H = DObjectHandle<ATop, DRepr>;
```

That is legal despite `obj<ATop,DRepr>` not existing (ticket 01), because members
of a class template are instantiated only when used, and `_native()` is never
called on such a handle. Confirmed 2026-09-06 by compiling `H<DFloat>` against
gcc 14.3 with the umbrella's flags — it is worth keeping a compile-only test so
the property cannot regress silently.

## Shape

Add the facet-parameterized recovery alongside `_native()`:

```cpp
template <typename AOther>
obj<AOther, DRepr> _native_as() const {
    return obj<AOther, DRepr>(static_cast<DRepr *>(this->_impl_handle()->opaque_data()));
}
```

Same construction as `_native()` after ticket 01 — from the slot's data pointer,
so `FacetImplType<AOther,DRepr>` resolves at compile time and the pointer is
re-read on every call. `_native()` then becomes `_native_as<AFacet>()`.

**Files:**
- Modify: `xo-facet/include/xo/facet/ObjectHandle.hpp`
- Test: `xo-facet/utest/objectmodel.test.cpp`

**Done when:**
- one `H<DRepr>` yields two different facets' `obj`s, each routing to that
  facet's implementation
- an `ATop`-keyed handle compiles without instantiating `_native()`
