# 03 — AReflectable, and the rotation that resolves an erased fomo object

Status: open
Type: feature
Milestone: reflectable2

The erased case: given `obj<AFacet>` whose `DRepr` is `DVariantPlaceholder`,
recover the concrete representation so reflection can continue through it.

This is the same problem `xo::reflect::SelfTagging` solves for non-fomo
objects, and the solution has the same shape — a capability a type opts into,
exposing one method that hands back its own `TaggedPtr`:

```bash
cat xo-reflect/include/xo/reflect/SelfTagging.hpp
grep -rn 'self_tp' xo-alloc/include xo-interpreter/include | head
```

## Shape

1. `AReflectable` gains its method: `self_tp(data) -> TaggedPtr`, named to match
   `SelfTagging::self_tp()`.

   `TaggedPtr`, not `TaggedRcptr`: fomo objects are arena-allocated, not
   refcounted, so validity belongs to the owning flywheel. Fine for synchronous
   traversal; callers must not retain one past the arena. State this in the
   header — it is the kind of constraint that is invisible at the call site.

2. `FomoTdx` overrides `TypeDescrExtra::most_derived_self_tp()` to rotate
   through `FacetRegistry` to `AReflectable` and return its `self_tp()`.

   The rotation already exists and is keyed on typeseq; nothing new is needed
   for runtime type recovery:

   ```bash
   grep -n 'try_variant' -A 6 xo-facet/include/xo/facet/FacetRegistry.hpp
   ```

   Cost is an arena-hashmap probe plus a virtual call, paid once per fomo node
   rather than per field.

3. Nothing else. `most_derived_self_tp` is a hook reflect ALREADY calls when
   descending into a struct member, so a `DRepr` holding erased fomo members is
   walked correctly with no change to printjson or to reflect:

   ```bash
   grep -n 'most_derived_self_tp' xo-reflect/include/xo/reflect/struct/StructMember.hpp \
                                  xo-reflect/src/reflect/struct/StructMember.cpp
   ```

## Testing

Use a throwaway D-type defined in xo-reflectable2's own utest, implementing
`AReflectable`. Deliberately NOT one of xo-object2's types: those are `05`, and
this ticket should not depend on it.

**Done when:**
- `most_derived_self_tp()` on an erased `obj<AFacet>` returns a `TaggedPtr`
  whose `td()` is the concrete representation's
- a struct holding an erased fomo member reflects through to that member's
  representation, via the existing `StructMember` path
- a type that does NOT implement `AReflectable` degrades visibly rather than
  crashing — decide and pin the behaviour (null `TaggedPtr` vs throw); the
  rotation returning failure is the observable, per `try_variant` returning
  null rather than throwing
- `xo-reflect` still unmodified
