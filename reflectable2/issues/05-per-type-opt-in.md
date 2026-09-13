# 05 — types opt in, one at a time

Status: open
Type: feature
Milestone: reflectable2

Implement `AReflectable` for the concrete representations, so real objects
render. This is the ticket that delivers the goal; `01`-`04` only make it
possible.

**One type at a time**, each an independent edit. Implementations live with the
D-type, not with the facet — already the convention:

```bash
find xo-*/include -name 'IPrintable_*.hpp' | head   # e.g. IPrintable_DString in xo-stringtable2
```

so each is `IReflectable_D<Foo>.hpp` beside its `IPrintable_D<Foo>.hpp`.

**The header is only half of it.** A `FacetImplementation` specialization is a
compile-time mapping; the rotation `03` added is a runtime lookup, so each type
also needs `FacetRegistry::register_impl<AReflectable, D<Foo>>()` in its
subsystem's `register_facets()`. Without it the type compiles, constructs and
erases, then fails the rotation at runtime -- see `03`'s outcome.

```bash
grep -n 'register_impl' xo-object2/src/object2/SetupObject2.cpp
```

**And usually a third piece: a json printer.** Reflection describes the
representation faithfully, so a box reflects as a struct; whether it should
READ as one in JSON is the printer's call. See the spec's "Where a type's json
printer lives" — `xo-printjson` was levelled below xo-object2 on 2026-09-12 so
that `SetupObject2::provide_json_printers` is possible at all.

So, per type, up to four things:

| | |
|---|---|
| `IReflectable_D<Foo>.hpp`/`.cpp` | generated from `idl/IReflectable_D<Foo>.json5` via `xo_add_genfacetimpl(... FACET_PKG xo_reflectable2 ...)`; delegates to a method ON the D-type, so the D-type gains `self_tp()` |
| `D<Foo>::reflect_self()` | a `StructReflector`, which must live INSIDE the type -- `REFLECT_MEMBER` takes `&D<Foo>::member_`, and those are private |
| `register_impl<AReflectable, D<Foo>>()` | in `SetupObject2::register_facets` |
| a `JsonPrinter` | in `SetupObject2::provide_json_printers`, only where the faithful rendering is not the one you want |

### Done: DFloat (2026-09-12)

Renders as `1.5`. Both halves pinned in
`xo-object2/utest/json_render.test.cpp`, and the pairing falsified: disabling
the printer registration leaves `{"_name_": "DFloat", "value": 1.5}` -- which
is reflection being honest, and the reason the printer exists.

One thing the facet needed, found here: an implementation's generated header
aliases only the types its FACET idl declares in `types:`, so
`xo-reflectable2/idl/Reflectable.json5` had to declare `TaggedPtr` before
`IReflectable_DFloat` would compile. Any later facet method with a non-builtin
return type needs the same.

#### The sweep earned its keep here

Adding `xo_dependency()` lines to `src/object2/CMakeLists.txt` is not enough:
xo-object2's `Config.cmake.in` was the HAND-maintained kind, so the two new
dependencies never reached the exported config. The umbrella build is fine
either way -- everything is in one cmake context -- and only the standalone
build breaks:

```
ld: cannot find -lxo_reflectable2: No such file or directory
ld: cannot find -lprintjson: No such file or directory
```

in xo-gc and xo-type, taking five more subsystems down as skipped. Fixed by
converting xo-object2 to the generated `@XO_FIND_DEPENDENCY_BLOCK@` form rather
than by adding two lines, so the parallel list cannot drift again:

```bash
grep -n 'find_dependency' ~/local/lib/cmake/xo_object2/xo_object2Config.cmake
```

**Every remaining type in this ticket touches the same file**, and most will
add no new dependency -- but the first one that does would have hit this. See
`.xo-backlog/generated-find-dependency/`.

Worth stating plainly because a green umbrella build looks like success: the
two-stage sweep is what distinguishes them, and its closing line
`--sweep ok (build and utest)` is the thing to read. Stage 2 ran here and
reported `0 failed` while stage 1 was broken.

## Candidates

```bash
ls xo-object2/include/xo/object2/D*.hpp
ls xo-stringtable2/include/xo/stringtable2/D*.hpp
```

Suggested order, chosen so nesting is proved early rather than last:

1. `DFloat` — a leaf; proves the path end to end with the least surface
2. `DList` — proves nesting, since its members are erased `obj<AGCObject>`
3. `DString`, `DUniqueString` (xo-stringtable2) — proves the facet works from a
   different subsystem than object2
4. `DInteger`, `DArray`, `DDictionary`, `DStruct`, `DBoolean`, `DRuntimeError`

Re-derive that list rather than trusting it; the D-type set moves.

## Python bindings

Each type may also gain a python binding via `ObjectHandle`, as a companion to
its opt-in. **Not a gate on the printjson goal** — decide per type. Where it
happens, the pyobject2 spec is the reference for how a representation is bound:
`.xo-backlog/pyobject2/spec.md`.

**Done when (per type):**
- `IReflectable_D<Foo>` exists beside that type's other facet implementations
- printjson emits JSON for an instance, reached from an erased `obj<AGCObject>`
- the type's existing tests still pass

**Done when (ticket):**
- a nested `DDictionary` of `DList`s of `DFloat`s renders as JSON in one call —
  the case the milestone exists for
- `Progress:` means of counting what is left, once the D-type list is settled:

```
Progress: comm -13 <(ls xo-object2/include/xo/object2/IReflectable_D*.hpp 2>/dev/null | sed 's/.*IReflectable_//') <(ls xo-object2/include/xo/object2/D*.hpp | sed 's/.*\///') | wc -l
```

  (adjust paths when the layout is known; a count that cannot be computed shows
  `[progress?]`, which is the intended failure mode — better than a stale
  number written into the ticket)
