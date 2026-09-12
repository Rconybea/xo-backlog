# 03 — AReflectable, and the rotation that resolves an erased fomo object

Status: done 2026-09-12
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

   Added as a `const_methods` entry in `xo-reflectable2/idl/Reflectable.json5`,
   NOT by editing the headers -- those are generated, and an edit to them is
   overwritten on the next build. `01` left the entry list empty with a TODO
   pointing here. The return type needs `<xo/reflect/TaggedPtr.hpp>` in the
   IDL's `includes`, which `01` also left empty:

   ```bash
   grep -n 'includes\|const_methods' xo-reflectable2/idl/Reflectable.json5
   grep -n 'includes' xo-printable2/idl/Printable.json5   # a designed facet, for the shape
   ```

   `TaggedPtr`, not `TaggedRcptr`: fomo objects are arena-allocated, not
   refcounted, so validity belongs to the owning flywheel. Fine for synchronous
   traversal; callers must not retain one past the arena. State this in the
   header — it is the kind of constraint that is invisible at the call site.

2. `FopTdx` overrides `TypeDescrExtra::most_derived_self_tp()` to rotate
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

   Either the thrown message names the D-type, or `FopTdx` catches and
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

## Outcome (2026-09-12)

Done. An ERASED fop now renders identical JSON to the typed one:

```
{"_name_": "DFopPoint", "x": 1.5, "y": -2.5}
```

which is the case that matters -- D-types hold erased members as a matter of
course, so what `02` delivered was the exception.

The failure message names the representation instead of a bare number:

```
FacetRegistry::variant failed :AFrom.tname xo::print::APrintable
  :ATo.tname AReflectable :DRepr.tseq 1 :DRepr.tname xo::ut::DOpaque
```

Fixed in `FacetRegistry::variant()` itself rather than caught and rethrown in
`FopTdx` (RC's call): the bare typeseq is a defect there, and every rotation in
the tree gets the readable failure. `FacetRegistry.hpp` already used
`TypeRegistry::id2name`, and nothing pinned the old text.

### Registration is explicit, and nothing said so

The thing this ticket got wrong. `FacetImplementation<AFacet,DRepr>` is a
COMPILE-TIME mapping; the rotation is a RUNTIME lookup, and the two are
connected only by an explicit call:

```bash
grep -n 'register_impl' -A 12 xo-facet/include/xo/facet/FacetRegistry.hpp
grep -rn 'register_facets' xo-object2/src/object2/SetupObject2.cpp
```

`valid_facet_implementation<>` is `consteval` and does static_asserts only -- it
registers nothing, despite reading like it might. A type with the
specialization but no `register_impl<>()` call compiles, constructs, erases, and
then fails the rotation at runtime.

`register_impl<>()` also calls `TypeRegistry::register_type<>()` for BOTH types,
which is what makes the improved message resolve a name. An unregistered type
renders as `_%sentinel%_` -- observed, before the registration calls were added.

**This lands on `05`**: opting a type in is `IReflectable_D<Foo>` *and* a
`register_impl<AReflectable, D<Foo>>()` in that subsystem's `register_facets()`,
not the header alone.

### The ticket's piece 2 was incomplete

It named `most_derived_self_tp` as the hook, true for the struct-member path
(`StructMember::get_tp()` calls it). But `PrintJson::print_tp` does NOT -- it
goes straight to the metatype switch. So `n_child()`/`child_tp()` had to rotate
as well, or a nested erased member prints correctly while a directly-handed one
prints `{}`. Both entry points are now pinned by their own test.

Counting deliberately does NOT rotate: a fop has a child exactly when its data
pointer is non-null, which is knowable without the registry. So a non-opted-in
representation reports 1 child and throws on the FETCH -- which is where
`print_generic_pointer` goes next, so printing still fails loudly.

### Detail: the test's not-opted-in type

`DOpaque` implements `APrintable` and nothing else -- the state every xo-object2
D-type is in before `05`. That costs xo-reflectable2's utest a dependency on
xo-printable2 (18, below 22, so legal) and the matching nix input.

It also turned up `.xo-backlog/xo-printable2/issues/03`: the convenience header
`<xo/printable2/Printable.hpp>` pulls `<xo/alloc2/Allocator.hpp>`, which
printable2 neither declares nor may depend on, alloc2 being levelled above it.
Worked around by including the `detail/` headers directly.

**Verified:** 21 assertions in 8 cases (reflectable2), 8 in 6 (printjson);
`git diff --stat xo-reflect/` empty; `nix-build ci.nix -A xo-reflectable2` and
`-A xo-printjson` both green with the check phase running.
