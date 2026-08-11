# 01 — xo-type's D-types have no APrintable facet, so printing one throws

Status: diagnosed
Type: bug

None of xo-type's D-types implements `APrintable`. Not "renders poorly" —
**there is no facet at all**, and asking for one throws.

```bash
ls xo-type/idl/ | grep -i printable                     # nothing
grep -rn 'APrintable' xo-type/src/ xo-type/include/     # nothing
```

`xo-type/idl/` carries `IGCObject_D*.json5` and `IType_D*.json5` for its six
D-types (`DArrayType`, `DAtomicType`, `DFunctionType`, `DListType`, `DTypeVar`,
`DTypeVarRef`) and no `IPrintable_D*` at all.

## What actually happens

`obj<AType>::to_facet<APrintable>()` reaches
`FacetRegistry::variant<APrintable,AType>`, which does not return an empty
`obj<>` — it throws (`xo-facet/include/xo/facet/FacetRegistry.hpp`):

```cpp
auto retval = try_variant<ATo>(from);
if (!retval)
    throw std::runtime_error(tostr("FacetRegistry::variant failed", ...));
```

So **any code path that prints a type terminates with an exception.**

## Confirmed by test, and the fault is one level down from where this said

`xo-expression2/utest/printable_render.test.cpp`, `DLocalSymtab-types-throws`,
added 2026-08-11, constructs a `DLocalSymtab` holding one `DAtomicType` and
renders it. Observed:

```
THREW: FacetRegistry::variant failed
       :AFrom.tseq 43 :AFrom.tname xo::scm::AType
       :ATo.tseq 41 :ATo.tname xo::print::APrintable :DRepr 16
```

The throw is real. **But this ticket named the wrong line for it.** It listed
two sites:

| site | field | actually |
|---|---|---|
| `DLocalSymtab::pretty_deprecated` | the `:types` loop, `(*types_)[i].to_facet<APrintable>()` | **succeeds** |
| `DTypename::pretty_deprecated` | `type_.to_facet<APrintable>()` | throws |

`types_` holds `DTypename`s, not `AType`s — `DLocalSymtab::append_type` wraps
each one (`DTypename::make(mm, name, type)`) before pushing it. `DTypename` has
an `IPrintable_DTypename.json5`, so the symtab's own facet lookup resolves
fine. The failure is inside the printer it then calls.

The wrong reading was plausible because both lines are spelled
`to_facet<APrintable>()` on an `obj<>` from a container, and because the symptom
— an exception while printing a symtab — is the same either way. It was reached
by grepping for `to_facet<APrintable>` in files that print types, without
checking what the container actually holds.

**Why it matters:** it moves the tolerate-vs-throw decision. Making
`DLocalSymtab` tolerant would fix nothing; the placeholder has to go in
`DTypename`, or the facet has to exist. `DLocalSymtab` converted as a pure
refactor for exactly that reason (2026-08-11).

**Nothing exercised either site before that test**, which is why this went
unnoticed:

```bash
grep -rn 'DLocalSymtab\|DTypename' --include=*.test.cpp xo-*/ | grep -v '/\.build/'
#   (no output, before 2026-08-11)
```

## Why it is not blocking the ppsink migration

RC's call, 2026-08-11: a refactor whose contract is "rendered output is
unchanged" cannot be blocked by output that does not exist. Recorded in
`.xo-backlog/xo-printable2/issues/01`, which first called this a blocker and was
corrected the same day.

`DLocalSymtab` / `DTypename` / `DLambdaExpr` can all convert with the type-valued
fields left exactly as they are — the phase-C expectations simply cannot cover
that path, because legacy has no rendering to pin against.

`DLocalSymtab` did exactly that on 2026-08-11: pure refactor, and its fixture
covers var-only cases. The `types_` path is pinned only as
`DLocalSymtab-types-throws` — legacy throws, ppsink renders, because ppsink
stops at `DTypename`'s phase-B stub instead of descending. **That asymmetry
closes when `DTypename` converts**, and that is the point at which this
decision has to be made rather than deferred: a converted `DTypename` either
reproduces the throw or renders a placeholder.

## Two ways to fix, and they are not equivalent

**1. Give xo-type an `APrintable` facet.** Six `IPrintable_D*.json5` beside the
existing IDLs, plus `pretty()` bodies. This is the real fix: types become
printable, and every enclosing printer improves for free. It is also the larger
job, and each of the six needs a rendering decided and pinned.

**2. Make the call sites tolerant.** `try_variant<>` plus a placeholder, so an
absent facet renders `<no APrintable>` instead of throwing. Small, and it stops a
debug dump of a symbol table taking down the process — but it entrenches the
absence rather than fixing it.

These compose: (2) now, (1) later. Doing (2) alone and calling it done would be
the failure mode.

**Why the ordering is not just about size** (RC, 2026-08-11): xo-type is itself
an in-flight refactor — schematika types are moving off `xo::reflect::TypeDescr`
onto the facet design, because TypeDescr commits a type to a *representation*
and the goal is to reason about types without specifying one. (Tolerable if
everything were JIT'd; not otherwise.) So (1) means deciding and pinning six
renderings against a D-type set that is still moving, and each pinned rendering
is a rebase later. (2) costs one placeholder and no pins.

The same fact reframes what phase C is pinning. `TypeRef` carries all three
tracks — `id_`, `type_` (`obj<AType>`), `td_` (`TypeDescr`) — and
`TypeRef::pretty` renders `:id` and `:td` and **never `type_`**. So every
phase-C expectation records only the departing half: 75 `:td` occurrences in
`xo-expression2/utest/printable_render.test.cpp`, 6 in xo-procedure2's. Not
wrong — phase C's contract is unchanged output — but those rebase when `td_`
goes, and that is a cost of this ticket landing late rather than of it landing
badly.

**Files:**
- `xo-type/idl/` — six `IGCObject_D*` and five `IType_D*`, no `IPrintable_*`
- `xo-facet/include/xo/facet/FacetRegistry.hpp` — `variant()` vs `try_variant()`
- `xo-expression2/src/expression2/DLocalSymtab.cpp` — the `:types` loop
- `xo-expression2/src/expression2/DTypename.cpp` — `type_.to_facet<APrintable>()`
- `.xo-backlog/xo-printable2/issues/01-aprintable-pretty-ppsink.md` — the
  migration, and the corrected "blocked" reading

**Done when:**
- ~~the throw is confirmed (or refuted) by an actual test, not a code-read~~
  — done 2026-08-11, `DLocalSymtab-types-throws`; it also relocated the fault
- a decision is recorded for the one call site that has it:
  `DTypename::pretty` — tolerate, or print
- if xo-type gains `APrintable`, each of the six D-types has a pinned rendering

## Notes

The gap is invisible from every direction that matters: no compile error (the
facet lookup is a runtime one), no test failure (nothing calls it), and no bad
output (the exception escapes before anything renders). The only reason it
surfaced was picking the next printer to convert and asking what its children
were.
