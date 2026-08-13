# 02 — xo-expression2 free of xo-indentlog

Status: fixed 2026-08-13
Type: refactor
Milestone: ppsink-migration
Progress: grep -rl '#include <xo/indentlog/' xo-expression2/ 2>/dev/null | grep -v '/\.build/' | wc -l

The goal is one command returning **rc=1**:

```bash
xo-deps --why=xo-expression2:xo-indentlog     # today: rc=0, one hop
```

Taken out of order, as a **prerequisite for
`.xo-backlog/xo-reader2/issues/02-reader2-free-of-indentlog.md`**: xo-expression2
is upstream of xo-reader2, so no amount of work inside xo-reader2 can make its
own `--why` query return rc=1 while this edge exists.

```bash
DEPS=$(xo-deps --deps-of=xo-reader2 --format=names -q | tr ' ' '\n' \
       | grep '^xo-' | grep -v '^xo-reader2$' | grep -v '^xo-indentlog$')
for d in $DEPS; do xo-deps --why=$d:xo-indentlog -q >/dev/null 2>&1 && echo "$d reaches xo-indentlog"; done
#   xo-expression2
```

Same shape as xo-object2, which had to wait for xo-printable2 and
xo-stringtable2 before its own declaration was the last one standing.

## It is nearly done already

**5 files**, and only **two** of them use anything:

```bash
for f in $(grep -rl '#include <xo/indentlog/' xo-expression2/ | grep -v '/\.build/'); do
  printf "%-52s %s\n" "$f" "$(grep -o '<xo/indentlog/[a-z/_0-9]*\.hpp>' $f | tr '\n' ' ')"
done
```

| file | legacy headers | uses |
|---|---|---|
| `src/expression2/SetupExpression2.cpp` | `scope.hpp` | 2 `scope`, 15 `xtag` |
| `src/expression2/DGlobalSymtab.cpp` | `scope.hpp` | 3 `scope` |
| `src/expression2/DTypename.cpp` | `print/quoted.hpp` | **nothing** |
| `src/expression2/DVariable.cpp` | `print/quoted.hpp` | **nothing** |
| `src/expression2/TypeRef.cpp` | `print/quoted.hpp`, `print/cond.hpp` | **nothing** |

The last three already call `xo::pp::quot` — phase E of
`.xo-backlog/xo-printable2/issues/01` converted their printers and left the
legacy includes behind. Verify per file rather than trusting the table:

```bash
for f in xo-expression2/src/expression2/{DTypename,DVariable,TypeRef}.cpp; do
  grep -nE '(^|[^:a-zA-Z_])(xtag|tostr|scope|quot|unq|cond|hex|pad|concat|refrtag)\(' $f | grep -v 'xo::pp::'
done
# only TypeRef.cpp:127, and that is a COMMENT recording what the legacy
# rendering was:  "Legacy said "null" here (cond(td_, td_, "null")); keep that,"
```

So three includes delete outright, and `print/cond.hpp` needs no ppsink
counterpart — nothing calls `cond`.

Note the `xtag`s in `SetupExpression2.cpp` are all of the shape
`xtag("X.tseq", typeseq::id<X>())`, identical to the blocks already converted in
`SetupProcedure2.cpp` and `SetupInterpreter2.cpp`.

## The conversion

The usual table (xo-object2 / xo-gc / xo-procedure2 / xo-tokenizer2):

| from | to |
|---|---|
| `#include <xo/indentlog/scope.hpp>` | `<xo/ppsink/scope.hpp>` + `<xo/ppsink/scope_macros.hpp>` |
| `scope log(XO_DEBUG(f))` | `scope log(XO_DEBUG_(f))` |
| `xo::scope` / `xo::xtag` | `xo::pp::scope` / `xo::pp::xtag` |

Namespace-scope using-declarations per `xo-arena/src/arena/mmap_util.cpp:16-20`.

`DGlobalSymtab.cpp:113` and `:259` pass a **second** argument to the scope ctor
(`scope log(XO_DEBUG(false), std::string_view(*var->name()))`), and `:196` spans
several lines. `XO_DEBUG_` mirrors legacy `XO_DEBUG` exactly, so the extra
arguments carry over unchanged — but this is the first conversion in the series
to use that form, so check the rendered banner rather than assuming.

Declared in the usual three places:

- `xo-expression2/src/expression2/CMakeLists.txt:80`
- `xo-expression2/cmake/xo_expression2Config.cmake.in:16`
- `pkgs/xo-expression2.nix:18,52`

Nothing upstream of xo-expression2 reaches xo-indentlog (the loop above,
run with `--deps-of=xo-expression2`, is silent), so its own declaration is the
only path.

## Verification

```bash
nix-build ci.nix -A xo-expression2 --no-out-link
xo-deps --users-of=xo-expression2 --format=names -q     # xo-interpreter2 xo-reader2
```

Both consumers still declare xo-indentlog themselves, so exposure should be
limited — but `xo-build --sweep` cannot see a dropped declaration, and the last
four subsystems in this series each surfaced at least one consumer that
`--sweep` called green. Build both under nix.

## Fixed 2026-08-13

```bash
xo-deps --why=xo-expression2:xo-indentlog     # rc=1
```

`Progress:` 5 -> 0, `subsystem-edges` re-captured (diff was exactly
`xo-indentlog xo-expression2`), `xo-build --sweep` →
`62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`, `nix-build` green
for xo-expression2 and both consumers.

The smoothest of the series, and the only one with **no surprises at all**: no
consumer exposed, no ADL, no rendering difference, and the file census was
exact. Three of the five files needed only their include deleted; the two real
conversions were 5 `scope`s and 15 `xtag`s.

The multi-argument scope form the ticket flagged as unverified —
`scope log(XO_DEBUG_(false), std::string_view(*var->name()))` at
`DGlobalSymtab.cpp:115`, `:198`, `:261` — compiled unchanged. `XO_DEBUG_` really
does mirror legacy `XO_DEBUG` including its trailing varargs.

This unblocks `.xo-backlog/xo-reader2/issues/02`, whose upstream check is now
silent.

## Suggested order

1. delete the three vestigial includes (`DTypename.cpp`, `DVariable.cpp`,
   `TypeRef.cpp`)
2. convert `SetupExpression2.cpp` and `DGlobalSymtab.cpp`
3. drop the three declarations; **re-capture** `subsystem-edges`
   (`.build/reconfigure`, then `.build/reconfigure --capture-subsystem-edges`,
   then reinstall xo-cmake so `xo-deps` reads the published copy) and confirm
   rc=1
4. then proceed to `.xo-backlog/xo-reader2/issues/02`, whose goal this unblocks

`Progress:` falls from 5 to 0 across steps 1 and 2.
