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
