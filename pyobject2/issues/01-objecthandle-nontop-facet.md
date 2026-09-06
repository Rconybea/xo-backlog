# 01 — DObjectHandle worked only for ATop, and nothing had ever instantiated it

Status: fixed 2026-09-06
Type: bug (latent)
Milestone: pyobject2

`DObjectHandle<AFacet,DRepr>` (`xo-facet/include/xo/facet/ObjectHandle.hpp`) did
not compile for any `AFacet` other than `ATop`, and the defect was invisible
because the template had no users:

```bash
grep -rn 'DObjectHandle' --include=*.cpp --include=*.hpp . \
    | grep -v 'ObjectHandle.hpp'      # empty before this ticket
```

Two defects, one per direction.

## 1. Narrowing in, by assignment

```cpp
obj<ATop> impl_obj = x;      // x is obj<AFacet,DRepr>
```

Both of `obj`'s converting constructors hold `AFacet` fixed and vary `DRepr`
(`sed -n '73,97p' xo-facet/include/xo/facet/obj.hpp`), so neither can change
facet. gcc:

```
error: conversion from 'obj<AComplex,DRectCoords>' to non-scalar type
       'obj<ATop,DVariantPlaceholder>' requested
```

Fixed by going through the variant constructor, which is sound because every
facet inherits `ATop` and nothing else — so `x.iface()` already IS an `ATop*`,
and the impl it names stays the concrete `IFacet_DRepr`:

```cpp
obj<ATop> impl_obj(static_cast<const ATop *>(x.iface()), x.opaque_data());
```

## 2. Recovery out, by reinterpret_cast

```cpp
return *reinterpret_cast<object_type *>(this->_impl_handle());
```

The slot holds `obj<ATop>`, whose stored interface bytes name whichever facet
`make_strong_ref` was handed. Reading them as `AFacet` routes through the wrong
vtable whenever the two differ. The obvious repair — have the slot hold
`obj<ATop,DRepr>` — is impossible: `ITop_Any` specializes only the variant,

```bash
grep -n 'FacetImplementation<ATop' xo-facet/include/xo/facet/top/ITop_Any.hpp
```

so `_native()` now constructs from the data pointer instead, which resolves
`FacetImplType<AFacet,DRepr>` at compile time:

```cpp
return object_type(static_cast<DRepr *>(this->_impl_handle()->opaque_data()));
```

This also makes the re-read discipline structural: no data pointer can be cached
across an allocating call, which matters once a moving collector records
relocations in the slot.

## Fixture note

`AComplex` in `xo-facet/utest/objectmodel.test.cpp` did not inherit `ATop` — an
oversight when `ATop` was introduced, since the fomo invariant is that every
facet inherits it. Fixed here (`_typeseq()` moves to `ATop` and gains `noexcept`;
`_drop()` delegates to the pre-existing `destruct_data`), which is what made the
bug reachable from a test at all.

**Verify:**

```bash
cd .build && make utest.facet && ./xo-facet/utest/utest.facet "objecthandle-nontop-facet"
```

**Done when:** met — a handle over a non-`ATop` facet round-trips: recovered
`obj` addresses the same representation, routes `AComplex` methods to the
`DRectCoords` implementation, and reports `typeseq::id<DRectCoords>()` through
the narrowed slot.
