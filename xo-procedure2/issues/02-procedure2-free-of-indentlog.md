# 02 — xo-procedure2 free of xo-indentlog

Status: fixed 2026-08-13
Type: refactor
Milestone: ppsink-migration
Progress: grep -rl '#include <xo/indentlog/' xo-procedure2/ 2>/dev/null | grep -v '/\.build/' | wc -l

The goal is one command returning **rc=1**:

```bash
xo-deps --why=xo-procedure2:xo-indentlog    # today: rc=0, one hop
```

Same shape as `.xo-backlog/xo-gc/issues/01-gc-free-of-indentlog.md` (fixed) and
`.xo-backlog/xo-object2/issues/01-object2-free-of-indentlog.md` (fixed), but
**much smaller** — and with one structural difference that the earlier two did
not have: a *public header* carries the legacy include.

## What xo-procedure2 uses it for

Legacy scope logging, nothing else. `scope.hpp` is the only legacy header
reached, in **5 files**:

```bash
for f in $(grep -rl '#include <xo/indentlog/' xo-procedure2/ | grep -v '/\.build/'); do
  printf "%-56s %s\n" "$f" "$(grep -o '<xo/indentlog/[a-z/_0-9]*\.hpp>' $f | tr '\n' ' ')"
done
```

Sizing, 2026-08-13 — re-derive rather than trusting these numbers (rule 5):

| file | `scope` | `xtag` |
|---|---|---|
| `src/procedure2/ObjectPrimitives.cpp` | 3 | 9 |
| `src/procedure2/SetupProcedure2.cpp` | 3 | 8 |
| `src/procedure2/PrimitiveRegistry.cpp` | 2 | 1 |
| `include/xo/procedure2/PrimitiveRegistry.hpp` | 1 | 2 |
| `utest/DPrimitive.test.cpp` | 0 | 0 |

No printer involvement — phase E of
`.xo-backlog/xo-printable2/issues/01-aprintable-pretty-ppsink.md` removed all of
that. This is 10 scope sites and 20 xtags.

## Its own declaration is the only path

```bash
DEPS=$(xo-deps --deps-of=xo-procedure2 --format=names -q | tr ' ' '\n' \
       | grep '^xo-' | grep -v '^xo-procedure2$' | grep -v '^xo-indentlog$')
for d in $DEPS; do xo-deps --why=$d:xo-indentlog -q >/dev/null 2>&1 && echo "$d reaches xo-indentlog"; done
# (no output)
```

so dropping xo-procedure2's own declaration is sufficient. It is declared in
**three** places, as everywhere in this tree:

- `xo-procedure2/src/procedure2/CMakeLists.txt:47` — `xo_dependency(${SELF_LIB} indentlog)`
- `xo-procedure2/cmake/xo_procedure2Config.cmake.in:12` — `find_dependency(indentlog)`
- `pkgs/xo-procedure2.nix:8,37` — argument + `propagatedBuildInputs`

xo-ppsink and xo-indentlog2 are **already declared** dependencies
(`xo-deps --deps-of=xo-procedure2 --format=names -q` lists both), so the
conversion adds nothing.

## The one thing that is different here: a public header

`include/xo/procedure2/PrimitiveRegistry.hpp:11` includes
`<xo/indentlog/scope.hpp>`, for the `scope log(XO_DEBUG(false))` inside the
`install_aux` **template** at `:76`. That include is therefore propagated to
every consumer of the header:

```bash
grep -rln 'procedure2/PrimitiveRegistry.hpp' --include=*.hpp --include=*.cpp xo-*/ | grep -v '/\.build/'
#   xo-reader2/src/reader2/ParserStateMachine.cpp
#   xo-reader2/include/xo/reader2/{ParserStateMachine,ParserConfig,ReaderConfig}.hpp
```

This is one of the paths by which xo-reader2 has been living on transitive
propagation (18 files, recorded in the phase-E notes). Removing it here does not
break xo-reader2 — `ParserStateMachine.cpp` includes `scope.hpp` and `tag.hpp`
itself — but it is the kind of change `xo-build --sweep` cannot see, so the
`nix-build` check below is not optional.

### ADL: expected to be quiet, and why

The template body's `xtag("name", pm->name())` is a **dependent** call
(`pm` has type `PrimitiveRepr *`), so it is resolved by ADL at the point of
instantiation — which is a xo-reader2 TU that *does* still have legacy
`xo::xtag` visible. That is exactly the shape that bit `SetupStringtable2.cpp`,
`ParserStack.cpp` and the `gp<Object>` case in `.xo-backlog/xo-alloc/issues/01`.

It should nevertheless be quiet here, because the associated namespaces of the
arguments contain no `xtag`:

```bash
grep -n 'name() const' xo-procedure2/include/xo/procedure2/DPrimitive.hpp
#   :120  std::string_view name() const noexcept { return name_; }
```

`const char[5]` associates nothing and `std::string_view` associates only
`std`, so ADL cannot reach `xo::xtag`. **Unverified until it compiles** — if it
does fire, the remedy is the established one: qualify at the call site, since a
using-declaration cannot suppress ADL.

Per the house rule for headers, the `using` goes **inside the function body**,
not at namespace scope, so it does not leak into includers.

## The conversion

Mechanical, per the pattern established in xo-object2 / xo-stringtable2 / xo-gc:

| from | to |
|---|---|
| `#include <xo/indentlog/scope.hpp>` | `<xo/ppsink/scope.hpp>` + `<xo/ppsink/scope_macros.hpp>` |
| `scope log(XO_DEBUG(f))` | `scope log(XO_DEBUG_(f))` |
| `xo::scope` / `xo::xtag` | `xo::pp::scope` / `xo::pp::xtag` |

`XO_DEBUG_` mirrors legacy `XO_DEBUG` exactly, including deriving the banner
from `__PRETTY_FUNCTION__` (`xo-ppsink/include/xo/ppsink/scope_macros.hpp`).
The `scope`/`xtag` facilities are in **xo-ppsink**, not xo-indentlog2 — ppsink
is where the arena-free front-end lives; xo-indentlog2 supplies the
arena-backed `PrettySink` behind it (see the header comment at
`xo-ppsink/include/xo/ppsink/scope.hpp:1-20`).

Three `src/` files take namespace-scope using-declarations, copied from
`xo-arena/src/arena/mmap_util.cpp:16-20`; the header takes block-scope ones.

## One vestigial include

`utest/DPrimitive.test.cpp:18` includes `<xo/indentlog/scope.hpp>` and carries
`using xo::scope;` at `:37`, but declares no `scope` and logs nothing — including
inside the `#ifdef OBSOLETE` block that
`.xo-backlog/xo-procedure2/issues/01-dprimitive-tests-disabled.md` covers. Both
lines delete outright; this file needs no ppsink include at all.

## Verification

`xo-build --sweep` will **not** catch a dropped declaration — the installed tree
keeps the headers findable. Every failure of that kind during phase E was found
by `nix-build` while `--sweep` was green, five times, and two of those five were
xo-procedure2 and xo-type surfacing when xo-object2 dropped **its** declaration.
So:

```bash
nix-build ci.nix -A xo-procedure2 --no-out-link
xo-deps --users-of=xo-procedure2 --format=names -q
#   xo-expression2 xo-interpreter2 xo-numeric xo-reader2
```

and build each consumer, since removing a propagated dependency is precisely
what exposes consumers that were relying on it transitively.

## Fixed 2026-08-13

```bash
xo-deps --why=xo-procedure2:xo-indentlog     # rc=1
```

All 5 files converted, the three declarations dropped, `subsystem-edges`
re-captured — the diff was exactly one line, `xo-indentlog xo-procedure2`.
`Progress:` 5 -> 0. `xo-build --sweep` →
`62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`; `nix-build` green
for xo-procedure2 and all four downstream consumers.

**ADL was quiet, as predicted.** Namespace-scope using-declarations in the three
`src/` files and block-scope ones in the template body sufficed; no call site
needed qualifying. The reasoning above held: `std::string_view` and `const
char[]` associate no namespace containing an `xtag`.

### Two consumers were exposed — one predicted, one not

**xo-reader2/utest/SchematikaParser.test.cpp (predicted).** It used `scope` and
`XO_DEBUG` with no include of its own, reaching them through
`PrimitiveRegistry.hpp`. Given an explicit `#include <xo/indentlog/scope.hpp>`
with a comment saying why — xo-reader2 still declares xo-indentlog, so
converting it belongs to that subsystem's own ticket, not this one.

**xo-numeric (not predicted, and not visible to `xo-build --sweep`).**
`src/numeric/NumericDispatch.cpp` and `src/numeric/SetupNumeric.cpp` both
included `<xo/indentlog/scope.hpp>` while xo-numeric **never declared
xo-indentlog** — its only declaration is a commented-out
`#find_dependency(indentlog)` at `xo_numericConfig.cmake.in:12`. They were
living entirely on xo-procedure2's propagation. `nix-build` caught it;
`--sweep` stayed green, exactly as this ticket warned.

Rather than give xo-numeric a declaration it never had, both files were
converted too. So **xo-numeric came free**:

```bash
xo-deps --why=xo-numeric:xo-indentlog        # rc=1
grep -rn '#include <xo/indentlog/' xo-numeric/ | grep -v '/\.build/'   # (none)
```

Worth generalising: a subsystem that reaches a legacy header only transitively
does not appear in **any** `--why` query for it, so removing a propagated
dependency is the only thing that finds it. Four such subsystems were declared
after phase E (xo-reader2, xo-interpreter2, xo-procedure2, xo-refcnt) — that
list was built from files that include legacy headers *and* declare the dep,
and so **could not have contained xo-numeric**. The census method, not the
census, was the gap.

### One rendering difference, measured and preserved

`NumericDispatch::dispatch` built its `DRuntimeError` text with legacy
`tosn(ss, ...)` into a `std::stringstream`. The natural replacement is
`xo::pp::tostr(...)`, which returns the string directly — and lets the
`std::stringstream` go, which the file never included `<sstream>` for anyway
(it arrived with indentlog).

They are **not** byte-identical: `tosn` is *to-stream-with-newline* and appends
one; `tostr` does not. Measured with a two-line scratch program rather than
assumed — legacy and ppsink agree on every byte including the color escapes,
and differ only in that trailing `\n`. It is preserved by passing `'\n'` as a
final argument, so the error text is unchanged.

**Open question, deliberately not settled here:** whether a `DRuntimeError`
message should carry a trailing newline at all. It looks like an artifact of
`tosn` being the convenient legacy call rather than a decision — but removing
it is a behaviour change, and this was a dependency refactor. The comment at
the call site points here.

## Suggested order

1. the 3 `src/procedure2/` files + the public header — this is what actually
   removes the propagated dep
2. `nix-build ci.nix -A xo-procedure2`, plus the four `--users-of` consumers
3. the vestigial include in `utest/DPrimitive.test.cpp`
4. drop the three declarations; **re-capture** `subsystem-edges` (it is
   generated, not authored — see `xo-object2/issues/01` for the procedure) and
   confirm `xo-deps --why=xo-procedure2:xo-indentlog` returns rc=1

`Progress:` counts files still including legacy indentlog anywhere under
`xo-procedure2/`, so it falls from 5 to 0 across steps 1 and 3.
