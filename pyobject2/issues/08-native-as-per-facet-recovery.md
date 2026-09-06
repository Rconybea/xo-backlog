# 08 — fold per-facet recovery into the handle, once a second binder exists

Status: deferred
Type: refactor
Milestone: pyobject2

A refactor, not a prerequisite. Recorded because the reasoning is easy to
rediscover badly — this ticket originally claimed ticket 05 depended on it,
which is backwards.

## What is actually needed, and when

`DObjectHandle<AFacet,DRepr>::_native()` returns `obj<AFacet,DRepr>` for the one
facet the handle is keyed to. That is sufficient for everything in this milestone
up to and including the first repr classes:

- a single-facet class keys the handle on the facet it uses, and `_native()`
  returns exactly the right `obj`
- plain representation members need no facet at all —
  `_native().data()->value()` reaches `DFloat::value()` whatever the key is

It first matters when ONE python class binds TWO facets — `List` / `Array`
wanting `ASequence` alongside `APrintable`. Even then it is not an enabler:
`ObjectHandleBase::_impl_handle()` is public, so a binder can construct the obj
it wants without any change to the handle:

```cpp
obj<AOther, DRepr>(static_cast<DRepr *>(h._impl_handle()->opaque_data()))
```

Confirmed 2026-09-06 by compiling exactly that, as a free function over
`H<DFloat>`, against gcc 14.3 with the umbrella's flags.

## The refactor

Once a second binder exists, that expression will be copy-pasted per binder,
carrying two things that should not be duplicated: the cast, and the rule that
the data pointer is re-read from the slot on every call rather than cached (it
is where a moving collector would record a relocation). Fold it into the handle:

```cpp
template <typename AOther>
obj<AOther, DRepr> _native_as() const {
    return obj<AOther, DRepr>(static_cast<DRepr *>(this->_impl_handle()->opaque_data()));
}
```

`_native()` then becomes `_native_as<AFacet>()`. NB the member-template form
above has not itself been compiled — only the free-function equivalent.

## Open question this settles

If the handle is keyed on `ATop` so one class can serve every facet
(`template <typename DRepr> using H = DObjectHandle<ATop, DRepr>;`, declared by
xo-pyfacet per ticket 03), then `AFacet` no longer varies and the two-parameter
form has no remaining job — `DObjectHandle<DRepr>` comes back. Decide that when
a second caller exists, not before.

The `ATop`-keyed alias is legal despite `obj<ATop,DRepr>` not existing (ticket
01), because members of a class template are instantiated only when used and
`_native()` is never called on such a handle. Compiled 2026-09-06; worth a
compile-only test if the alias is adopted, since nothing else would catch a
regression.

**Files (when taken up):**
- Modify: `xo-facet/include/xo/facet/ObjectHandle.hpp`
- Test: `xo-facet/utest/objectmodel.test.cpp`

**Done when:**
- one handle yields two different facets' `obj`s, each routing to that facet's
  implementation, with the cast written once
