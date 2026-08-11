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

So **any code path that prints a type terminates with an exception.** Two are
known, both in xo-expression2:

| site | field |
|---|---|
| `DLocalSymtab::pretty_deprecated` | the `:types` loop, `(*types_)[i].to_facet<APrintable>()` |
| `DTypename::pretty_deprecated` | `type_.to_facet<APrintable>()` |

**Nothing exercises either**, which is why this has gone unnoticed:

```bash
grep -rn 'DLocalSymtab\|DTypename' --include=*.test.cpp xo-*/ | grep -v '/\.build/'
#   (no output)
```

Measured 2026-08-11. **Unverified:** that the throw is what happens at runtime
rather than what the code reads as — no test constructs a `DLocalSymtab` with a
non-empty `types_`, so this is a code-read, not an observation. Worth confirming
with a two-line test before designing the fix, since "it throws" and "it renders
nothing" call for different remedies.

## Why it is not blocking the ppsink migration

RC's call, 2026-08-11: a refactor whose contract is "rendered output is
unchanged" cannot be blocked by output that does not exist. Recorded in
`.xo-backlog/xo-printable2/issues/01`, which first called this a blocker and was
corrected the same day.

`DLocalSymtab` / `DTypename` / `DLambdaExpr` can all convert with the type-valued
fields left exactly as they are — the phase-C expectations simply cannot cover
that path, because legacy has no rendering to pin against.

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

**Files:**
- `xo-type/idl/` — six `IGCObject_D*` and five `IType_D*`, no `IPrintable_*`
- `xo-facet/include/xo/facet/FacetRegistry.hpp` — `variant()` vs `try_variant()`
- `xo-expression2/src/expression2/DLocalSymtab.cpp` — the `:types` loop
- `xo-expression2/src/expression2/DTypename.cpp` — `type_.to_facet<APrintable>()`
- `.xo-backlog/xo-printable2/issues/01-aprintable-pretty-ppsink.md` — the
  migration, and the corrected "blocked" reading

**Done when:**
- the throw is confirmed (or refuted) by an actual test, not a code-read
- a decision is recorded for each of the two call sites: tolerate, or print
- if xo-type gains `APrintable`, each of the six D-types has a pinned rendering

## Notes

The gap is invisible from every direction that matters: no compile error (the
facet lookup is a runtime one), no test failure (nothing calls it), and no bad
output (the exception escapes before anything renders). The only reason it
surfaced was picking the next printer to convert and asking what its children
were.
