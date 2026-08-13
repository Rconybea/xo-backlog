# 01 — xo-gc free of xo-indentlog

Status: fixed 2026-08-13
Type: refactor
Milestone: ppsink-migration
Progress: grep -rl '#include <xo/indentlog/' xo-gc/ 2>/dev/null | grep -v '/\.build/' | wc -l

The goal is one command returning **rc=1**:

```bash
xo-deps --why=xo-gc:xo-indentlog     # today: rc=0
```

Same shape as `.xo-backlog/xo-object2/issues/01-object2-free-of-indentlog.md`,
which is `fixed`, but ~20x the call sites and with **no printer involvement at
all**.

## What xo-gc actually uses it for

Nothing left over from `pretty_deprecated` — phase E of
`.xo-backlog/xo-printable2/issues/01` removed all of that. Every remaining
include is legacy **logging**:

```bash
for f in $(grep -rl '#include <xo/indentlog/' xo-gc/ | grep -v '/\.build/'); do
  printf "%-40s %s\n" "$f" "$(grep -o '<xo/indentlog/[a-z/_0-9]*\.hpp>' $f | tr '\n' ' ')"
done
```

reports only `scope.hpp`, `print/tag.hpp`, and one `print/array.hpp`.

Sizing, 2026-08-13 — re-derive rather than trusting these numbers (rule 5; the
`Progress:` count above is the means):

| | files | `scope` sites | `xtag` sites |
|---|---|---|---|
| `src/gc/` (library) | 6 | 33 | 134 |
| `utest/` | 9 | 20 | 78 |

Concentrated in four library files: `GCObjectStore.cpp` (14 scopes / 54 xtags),
`DX1Collector.cpp` (7/29), `MutationLogStore.cpp` (7/24),
`DX1CollectorIterator.cpp` (3/21).

## Its own declaration is the only path

```bash
DEPS=$(xo-deps --deps-of=xo-gc --format=names -q | tr ' ' '\n' | grep '^xo-' | grep -v '^xo-gc$')
for d in $DEPS; do xo-deps --why=$d:xo-indentlog -q >/dev/null 2>&1 && echo "$d reaches xo-indentlog"; done
# (no output)
```

Nothing upstream of xo-gc reaches xo-indentlog, so removing xo-gc's own
declaration is sufficient — unlike xo-object2, which additionally had to wait
for xo-printable2 and xo-stringtable2.

Declared in **three** places, as everywhere in this tree:

- `xo-gc/src/gc/CMakeLists.txt:38` — `xo_dependency(${SELF_LIB} indentlog)`
- `xo-gc/cmake/xo_gcConfig.cmake.in:8` — `find_dependency(indentlog)`
- `pkgs/xo-gc.nix` — argument + `propagatedBuildInputs`

## The conversion

Mechanical, per the pattern established in xo-object2 / xo-stringtable2:

| from | to |
|---|---|
| `#include <xo/indentlog/scope.hpp>` | `<xo/ppsink/scope.hpp>` + `<xo/ppsink/scope_macros.hpp>` |
| `#include <xo/indentlog/print/tag.hpp>` | `<xo/ppsink/tag.hpp>` |
| `tostr(...)` sites | also `<xo/ppsink/tostr.hpp>` |
| `scope log(XO_DEBUG(f))` | `scope log(XO_DEBUG_(f))` |
| `xo::scope` / `xo::xtag` | `xo::pp::scope` / `xo::pp::xtag` |

`XO_DEBUG_` mirrors legacy `XO_DEBUG` exactly, including deriving the banner
from `__PRETTY_FUNCTION__` (`xo-ppsink/include/xo/ppsink/scope_macros.hpp`).

Import style, both idioms already in use in this tree:

- namespace-scope using-declarations, copied from
  `xo-arena/src/arena/mmap_util.cpp:16-20` — works once the legacy include is
  **gone** from the file
- qualified at the call site where a legacy header must remain

### Two traps, both already paid for elsewhere

**ADL.** `xo::pp::xtag("k", "literal")` with `const char[]` arguments still
resolves ambiguously against a legacy overload, and **a using-declaration
cannot suppress ADL**. Qualify at the call site. Bit three times already
(`SetupStringtable2.cpp`, `ParserStack.cpp`, and the `gp<Object>` case in
`.xo-backlog/xo-alloc/issues/01`). The `tostr`/`hex` sites in
`GcosTestutil.cpp`, `GCObjectStore.test.cpp`, `MutationLogStore.test.cpp` and
`random_allocs.cpp` are the likeliest to hit it.

**`xo-build --sweep` will not catch a dropped declaration.** The installed tree
keeps the headers findable. Every failure of this kind during phase E was found
by `nix-build` while `--sweep` was green — five times. Verify with
`nix-build ci.nix -A xo-gc --no-out-link`, and note xo-gc has **downstream**
consumers, so check them too:

```bash
xo-deps --users-of=xo-gc --format=names -q
```

Removing a propagated dependency exposes consumers that were relying on it
transitively — that is exactly how xo-type and xo-procedure2 surfaced when
xo-object2 dropped its own.

## One vestigial include

`xo-gc/utest/Collector.test.cpp` includes `<xo/indentlog/print/array.hpp>` and
uses nothing from it — its only non-scope/xtag call is `tostr(xtag(...))` at
`:446`. Deletes outright.

## Fixed 2026-08-13

```bash
xo-deps --why=xo-gc:xo-indentlog     # rc=1
```

All 15 files converted (6 library + 9 utest), the three declarations dropped,
`subsystem-edges` re-captured (the diff was exactly `xo-indentlog xo-gc`).
`Progress:` 15 -> 0. xo-gc suite unchanged at **8578 assertions in 20 test
cases**; `xo-build --sweep` →
`62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`; `nix-build` green
for xo-gc and all three downstream consumers (xo-expression2, xo-interpreter2,
xo-reader2 — none was exposed, since each already declares indentlog itself).

### What differed from the earlier subsystems

**ADL never fired.** Every earlier conversion needed at least one call-site
qualification; here namespace-scope using-declarations sufficed throughout. The
difference is that no legacy `tag.hpp` reaches these TUs transitively any more,
so there is no competing overload to be ambiguous with. Worth knowing: the
trap is a property of *what else the TU includes*, not of the call.

**Catch2's `INFO()` needs the ostream bridge.** `INFO(tostr(xtag(...)))`
streams to an ostream, so a file using it needs
`<xo/ppsink/tag_ostream.hpp>` alongside `tag.hpp` — otherwise
`no match for 'operator<<' ... const xo::pp::tag_impl<...>`. Five of the nine
utest files.

**Not every file is inside `namespace xo`.** Six of the nine open
`namespace ut {` (or `namespace utest {` in `random_allocs.cpp`) with **no
enclosing `namespace xo`**, and they carried explicit `using xo::scope;` /
`using xo::xtag;` / `using xo::tostr;` declarations. Those had to be rewritten
to `xo::pp::`, not merely supplemented — a transform that inserts a using-block
at a `namespace xo {` anchor silently does nothing useful for them.

### A process note

The first attempt reported an assertion failure naming the wrong cause: the
script asserted on leftover legacy includes, but the actual problem was the
missing `namespace xo {` anchor further down. Replacing the assert with
per-file diagnostics (`!! <file>: <reason>`) found it immediately. **A bulk
transform over a file set should report per file and continue, not assert on
the first anomaly** — the assert names where it stopped, not what is wrong.

## Suggested order

1. the 6 `src/gc/` files — this is what actually removes the propagated dep
2. `nix-build ci.nix -A xo-gc`, plus the `--users-of` list
3. the 9 `utest/` files
4. drop the three declarations; **re-capture** `subsystem-edges` (it is
   generated, not authored — see `xo-object2/issues/01` for the procedure) and
   confirm `xo-deps --why=xo-gc:xo-indentlog` returns rc=1

`Progress:` counts files still including legacy indentlog anywhere under
`xo-gc/`, so it falls from 15 to 0 across steps 1 and 3.
