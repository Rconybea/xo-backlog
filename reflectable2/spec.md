# reflectable2 — reflection for faceted objects

Status: open
Type: spec
Milestone: reflectable2

Let c++ code interrogate a fomo object (`obj<AFacet,DRepr>`) at runtime without
knowing its representation, by giving reflection a way to reach through the
facet to the underlying `DRepr`. The first consumer is `xo-printjson`.

## Purpose

`xo-reflect` exists to carry information to runtime that the c++ compiler would
otherwise discard. Fomo objects are ordinary c++ objects that reflection simply
cannot see into, so any reflection-driven facility — serialization, inspection,
a generic walker — stops at the facet boundary.

**This is unrelated to the `xo-type` / `TypeDescr` question.** That refactor
moves *schematika* type representation off `TypeDescr`, because committing a
schematika type to a c++ representation makes a schematika compiler impossible
without invoking the c++ compiler. Fomo objects are already represented in c++;
nothing here asks `TypeDescr` to describe a schematika type. Recorded because
the two look superficially opposed.

### The longer-range reason

Getting printjson + reflect working over fomo makes xo state reachable from a
browser. Visualizations and animations could then be grounded in actual xo
state rather than in a parallel model maintained alongside it. Parallel models
drift from what they depict; that is the cost being avoided, and it is why this
is worth more than "JSON for fomo objects".

## What printjson actually needs

Very little, which is what makes this tractable. It dispatches on metatype and
walks children:

```bash
sed -n '150,175p' xo-printjson/src/printjson/PrintJson.cpp   # the Metatype switch
grep -n 'n_child\|get_child\|struct_member_name' xo-printjson/src/printjson/PrintJson.cpp
```

So the whole question is where a fomo object's children come from.

## Design

A new subsystem `xo-reflectable2`, levelled just above `xo-reflect`, providing
three pieces.

**1. `AReflectable`** — an opt-in capability facet whose single method returns a
`TaggedPtr` for the object's concrete representation. Named `self_tp()` to
match `xo::reflect::SelfTagging::self_tp()`, which solves the same problem for
non-fomo objects:

```bash
cat xo-reflect/include/xo/reflect/SelfTagging.hpp
grep -rn 'self_tp' xo-alloc/include xo-interpreter/include | head    # implementors
```

`TaggedPtr`, not `TaggedRcptr`: fomo objects are arena-allocated rather than
refcounted, so validity is the owning flywheel's. Fine for synchronous
traversal; a caller must not retain one past the arena.

**2. `FomoTdx : TypeDescrExtra`** — overrides `most_derived_self_tp()` to rotate
the object through `FacetRegistry` to `AReflectable`, then return what
`self_tp()` gives. This is the erased path, and it lands on a hook reflect
ALREADY calls when descending into a struct member:

```bash
grep -n 'most_derived_self_tp' xo-reflect/include/xo/reflect/TypeDescrExtra.hpp \
                               xo-reflect/include/xo/reflect/struct/StructMember.hpp
```

That is what makes nested fomo members work with no change to printjson: a
`DRepr` holding `obj<AGCObject>` members (routine — see `DDictionary`, `DList`,
`DArray` in xo-object2) is walked by the existing struct machinery, which asks
the hook what is really there.

**3. `EstablishTdx<obj<AFacet,DRepr>>`** — a fomo object reflects as
`mt_pointer` with one child, its representation, which is what it is: a facet
plus a data pointer. No new `Metatype`; printjson's existing
`print_generic_pointer` handles it. When `DRepr` is concrete rather than
`DVariantPlaceholder`, this shortcuts to `Reflect::require<DRepr>()` — the
static case is the fast path of the erased one, not a separate feature.

## What is NOT modified

**`xo-reflect` is untouched.** `TypeDescrExtra` and `EstablishTdx` are public
extension points, so all three pieces above can be supplied from outside. This
is a requirement, not an accident: hosting them inside reflect would create a
reflect -> facet dependency, inherited by every reflect dependent. Count the
ones that would newly acquire xo-facet:

```bash
for s in $(xo-deps --users-of=xo-reflect --format=names -q); do
    xo-deps --why=$s:xo-facet -q >/dev/null || echo "$s"
done
```

As of 2026-09-12 that set includes `xo-unit`, `xo-ratio`, `xo-ordinaltree`,
`xo-process`, `xo-websock` and the legacy v1 stack — none of which have any use
for the facet object model.

**`xo-printjson` is barely modified**: one new entry point,
`print_obj(obj<AReflectable>, ostream*)`, beside the existing
`print_obj(rp<SelfTagging>, ostream*)`. Nested fomo members need no change at
all.

## Why opt-in, and why a separate subsystem

The alternative considered was requiring every facet to provide a `_repr_tp()`
method, which would make all fomo objects reflectable with no opt-in. Rejected
on two grounds:

- it puts a reflection concern into every abstract facet, including `AEquable`,
  `AHashable`, `AAllocator`, `ANumeric`, which have no use for one. Means of
  counting the edit:

  ```bash
  find xo-*/include -name 'A[A-Z]*.hpp' | wc -l        # abstract facets
  find xo-*/include -name 'I[A-Z]*_*.hpp' | wc -l      # facet implementations
  ```

- it requires xo-facet to be levelled ABOVE xo-reflect, inverting the current
  order and dragging the subsystems currently between them:

  ```bash
  grep -n '^xo-facet$\|^xo-reflect$' xo-cmake/etc/xo/subsystem-list
  for s in $(xo-deps --users-of=xo-facet --format=names -q); do
      grep -n "^$s$" xo-cmake/etc/xo/subsystem-list; done | sort -n
  ```

Opt-in is also what `xo-printjson` already requires of non-fomo objects: a type
that wants JSON inherits `SelfTagging` and says so. Requiring `AReflectable` of
fomo types is the same bargain, not a new imposition.

The subsystem follows the established shape for a cross-cutting capability
facet — one facet, one subsystem — as `APrintable`/`xo-printable2`,
`AEquable`/`xo-equable2`, `AHashable`/`xo-hashable2` already do.

## Levelization

`xo-reflectable2` sits above `xo-reflect` and below the subsystems whose types
opt in, so each type implements the facet in its own home. That is already the
convention for facet implementations — they live with the D-type, not with the
facet:

```bash
find xo-*/include -name 'IPrintable_*.hpp' | head    # e.g. in xo-stringtable2, not xo-printable2
grep -n '^xo-reflect$\|^xo-stringtable2$\|^xo-object2$' xo-cmake/etc/xo/subsystem-list
```

## Scope

v1 is reflection plus JSON. Per-type opt-in proceeds one type at a time across
`xo-object2` and `xo-stringtable2`. Python bindings for each type (via
`ObjectHandle`) are a possible companion to each opt-in, NOT a gate on the
printjson goal.

The browser/animation use case is the motivation, not v1 scope: nothing here
ships a transport or a viewer.

## Tickets

`01` scaffold, `02` static path, `03` facet + rotation, `04` printjson entry
point, `05` per-type opt-in. Each is independently reviewable; `02` already
produces JSON via a hand-built `TaggedPtr`, so output is visible before `04`.
