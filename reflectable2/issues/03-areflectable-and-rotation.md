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

   **Use the THROWING form** -- `variant<AReflectable>`, not
   `try_variant<AReflectable>`. A representation that has not opted in is a
   programming error, not a data-dependent condition, so it should not
   silently render as empty. Both forms already exist; this is a choice
   between them, not new machinery:

   ```bash
   sed -n '138,162p' xo-facet/include/xo/facet/FacetRegistry.hpp   # variant() throws
   ```

   The stock message names `AFrom`/`ATo` by typeseq AND name, but reports the
   representation as a bare typeseq number (`xtag("DRepr", from._typeseq())`).
   For this failure the representation is the thing the reader needs, so
   resolve it:

   ```bash
   grep -n 'id2name' xo-facet/include/xo/facet/TypeRegistry.hpp
   ```

   Either the thrown message names the D-type, or `FomoTdx` catches and
   rethrows with it added. "DRepr 47 does not implement AReflectable" makes a
   reader go look up 47.

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
- a type that does NOT implement `AReflectable` **throws**, and the exception
  names the representation by name rather than by typeseq number
- the partial-output consequence is understood and pinned by a test: throwing
  mid-traversal leaves whatever the consumer had already written. For printjson
  that means a truncated JSON document plus an exception, which is the intended
  trade -- a programming error surfaces loudly rather than producing
  well-formed JSON that quietly omits an object
- `xo-reflect` still unmodified
