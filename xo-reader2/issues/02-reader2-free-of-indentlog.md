# 02 — xo-reader2 free of xo-indentlog

Status: fixed 2026-08-13
Type: refactor
Milestone: ppsink-migration
Progress: grep -rl '#include <xo/indentlog/' xo-reader2/ 2>/dev/null | grep -v '/\.build/' | wc -l

The goal is one command returning **rc=1**:

```bash
xo-deps --why=xo-reader2:xo-indentlog     # today: rc=0
```

The **largest** of the series (after xo-object2, xo-gc, xo-procedure2,
xo-tokenizer2, xo-expression2) and, with xo-interpreter2, one of the last two.

## Blocked until xo-expression2 was done

xo-expression2 is upstream, and reached xo-indentlog until 2026-08-13:

```bash
DEPS=$(xo-deps --deps-of=xo-reader2 --format=names -q | tr ' ' '\n' \
       | grep '^xo-' | grep -v '^xo-reader2$' | grep -v '^xo-indentlog$')
for d in $DEPS; do xo-deps --why=$d:xo-indentlog -q >/dev/null 2>&1 && echo "$d reaches xo-indentlog"; done
# before: xo-expression2      after .xo-backlog/xo-expression2/issues/02: (no output)
```

so xo-reader2's own declaration is now the only path — but **only since that
ticket closed**. Re-run the loop before starting; if it is not silent, this
ticket cannot reach rc=1 no matter how many files are converted.

## Sizing

**19 files**, all reaching `scope.hpp` except where noted. Re-derive rather than
trusting the table (rule 5):

```bash
for f in $(grep -rl '#include <xo/indentlog/' xo-reader2/ | grep -v '/\.build/' | sort); do
  printf "%-52s %s\n" "${f#xo-reader2/}" "$(grep -o '<xo/indentlog/[a-z/_0-9]*\.hpp>' $f | tr '\n' ' ')"
done
```

| file | `scope` | `xtag` | `tostr` |
|---|---|---|---|
| `src/reader2/ParserStateMachine.cpp` | 17 | 65 | 20 |
| `utest/SchematikaParser.test.cpp` | 34 | 39 | 1 |
| `src/reader2/DProgressSsm.cpp` | 18 | 23 | 5 |
| `src/reader2/SetupReader2.cpp` | 2 | 20 | 0 |
| `src/reader2/DDefineSsm.cpp` | 15 | 8 | 0 |
| `src/reader2/SchematikaReader.cpp` | 1 | 11 | 0 |
| `src/reader2/DIfElseSsm.cpp` | 10 | 3 | 0 |
| `src/reader2/DExpectExprSsm.cpp` | 7 | 9 | 0 |
| `src/reader2/DParenSsm.cpp` | 5 | 7 | 0 |
| `src/reader2/DSchematikaParser.cpp` | 1 | 6 | 1 |
| `src/reader2/DApplySsm.cpp` | 5 | 3 | 0 |
| `src/reader2/DSequenceSsm.cpp` | 3 | 2 | 0 |
| `src/reader2/DExpectSymbolSsm.cpp` | 2 | 2 | 0 |
| `src/reader2/DExpectTypeSsm.cpp` | 2 | 1 | 0 |
| `src/reader2/DGlobalEnv.cpp` | 1 | 2 | 0 |
| `src/reader2/DDeftypeSsm.cpp` | 1 | 1 | 0 |
| `src/reader2/DToplevelSeqSsm.cpp` | 1 | 1 | 0 |
| `src/reader2/DExpectQLiteralSsm.cpp` | 0 | 0 | 0 |
| `src/reader2/DExpectFormalArglistSsm.cpp` | 0 | 0 | 0 |
| **total** | **125** | **203** | **27** |

Bigger than xo-gc (53 scopes / 212 xtags / 15 files), and the concentration is
different: two files carry 40% of it.

### Three vestigial includes

The last two rows use nothing — `DExpectFormalArglistSsm.cpp` is already on
`xo::pp::concat`, phase E just left its include behind. `DProgressSsm.cpp`
additionally includes `<xo/indentlog/print/cond.hpp>` and never calls `cond`.
All three delete outright, with no ppsink counterpart needed.

### The count is a lower bound — check by USE, not by include

This is the third subsystem in a row where the include-based census missed
files. In xo-tokenizer2 it missed two of eight, *inside the subsystem*. The
reliable enumeration:

```bash
for f in $(grep -rlE '\b(xtag|tostr|scope|refrtag|tosn|quot|hex|concat|unq|pad)\b' \
           --include=*.hpp --include=*.cpp xo-reader2/ | grep -v '/\.build/'); do
  grep -q '#include <xo/indentlog/' $f || echo "$f"
done
```

Run 2026-08-13 it lists 7 files, and **all 7 are already on ppsink or are
comments** — `readerreplxx.cpp`, `ParserResult.cpp`, `ParserStack.cpp`,
`printable_render.test.cpp` carry `xo/ppsink/` includes and `xo::pp::` names;
`DSchematikaParser.hpp:337`, `ParserStateMachine.hpp:181` and
`DExpectQDictSsm.cpp:265` match only inside comments. So for **this** subsystem
the 19 happens to be exact. Verify it again after the conversion — the same
command should return only ppsink users.

## The conversion

The usual table:

| from | to |
|---|---|
| `#include <xo/indentlog/scope.hpp>` | `<xo/ppsink/scope.hpp>` + `<xo/ppsink/scope_macros.hpp>` |
| `#include <xo/indentlog/print/tag.hpp>` | `<xo/ppsink/tag.hpp>` |
| `#include <xo/indentlog/print/tostr.hpp>` | `<xo/ppsink/tostr.hpp>` |
| `os << xtag(...)` | also `<xo/ppsink/tag_ostream.hpp>` |
| `scope log(XO_DEBUG(f))` | `scope log(XO_DEBUG_(f))` |
| `xo::scope` / `xo::xtag` / `xo::tostr` | `xo::pp::` equivalents |

Every one of the 19 files opens `namespace xo {` (`DDefineSsm.cpp:17` with a
leading space; `utest/SchematikaParser.test.cpp` nests `namespace ut {` inside
it), so namespace-scope using-declarations work throughout — the
`xo-arena/src/arena/mmap_util.cpp:16-20` idiom. Only `DSchematikaParser.cpp`
already carries explicit `using xo::tostr; using xo::xtag;`, which must be
**rewritten** rather than supplemented.

Four files stream tags to an ostream and need the `tag_ostream.hpp` bridge:

```bash
grep -rlE '<< *xtag\(' --include=*.cpp xo-reader2/src | grep -v '/\.build/'
#   DDefineSsm.cpp DExpectExprSsm.cpp DParenSsm.cpp DSchematikaParser.cpp
```

Rewriting those `print(std::ostream &)` methods onto a `PpSink &` is the
`ostream-containment` milestone, not this ticket.

### A parked block that will not be converted

`DDefineSsm.cpp:64-338` is `#ifdef NOT_YET` and contains a legacy
`pretty_print(const xo::print::ppindentinfo &)` with `refrtag(...)` at `:336`.
It is **already stale** — `ppindentinfo` was deleted in phase E of
`.xo-backlog/xo-printable2/issues/01`, so the block could not compile even if
`NOT_YET` were defined. Left alone deliberately: it is parked work, and
resurrecting it is a separate decision from removing a dependency. Worth
recording so nobody reads its survival as evidence it still builds.

### ADL is the risk here, more than anywhere before

125 scopes across 19 TUs, several of which include headers from subsystems that
still carry legacy `tag.hpp` (xo-interpreter2 is downstream, but xo-reader2's
own `ParserStateMachine.cpp` includes `<xo/procedure2/PrimitiveRegistry.hpp>`
and much else). A using-declaration **cannot** suppress ADL; the remedy is to
qualify at the call site. Four earlier conversions were bitten
(`SetupStringtable2.cpp`, `ParserStack.cpp`, the `gp<Object>` case in
`.xo-backlog/xo-alloc/issues/01`), and xo-gc was clean only because no legacy
`tag.hpp` reached its TUs at all — the trap is a property of *what else the TU
includes*, not of the call.

## Verification — this subsystem actually has tests

Unlike xo-tokenizer2, xo-reader2 has real coverage, and it is the safety net for
everything below it:

```bash
ctest --test-dir xo-reader2/.build
./xo-reader2/.build/utest/utest.reader2 | tail -3
#   1837 assertions in 33 test cases   (2026-08-13, before this ticket)
```

That number must not move. Then:

```bash
nix-build ci.nix -A xo-reader2 --no-out-link
xo-deps --users-of=xo-reader2 --format=names -q     # xo-interpreter2
```

`xo-build --sweep` cannot see a dropped declaration — every subsystem in this
series surfaced a consumer that `--sweep` called green. xo-interpreter2 is the
only consumer, and it still declares xo-indentlog itself, so exposure should be
small; build it under nix regardless. Build with `--with-examples`:
`example/readerreplxx/readerreplxx.cpp` is already on ppsink, but it is invisible
to a build that omits examples.

## Fixed 2026-08-13

```bash
xo-deps --why=xo-reader2:xo-indentlog     # rc=1
```

`Progress:` 19 -> 0, `subsystem-edges` re-captured (diff exactly
`xo-indentlog xo-reader2`), `xo-build --sweep` →
`62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`, `nix-build` green
for xo-reader2 and xo-interpreter2. The suite is **unchanged at 1837 assertions
in 33 test cases** — the number the ticket said must not move.

### One subsystem left

```bash
for s in $(xo-build --list | tr ' ' '\n' | grep '^xo-' | sort -u); do
  [ -f "$s/CMakeLists.txt" ] || continue; [ "$s" = xo-indentlog ] && continue
  xo-deps --why=$s:xo-indentlog -q >/dev/null 2>&1 && echo "$s"
done
#   xo-interpreter2
```

**xo-interpreter2, 8 files.** After that, nothing in the tree reaches
xo-indentlog.

### The predicted ADL trouble did not happen

The ticket called ADL "the risk here, more than anywhere before" — 125 scopes
across 19 TUs. **Zero call sites needed qualifying.** Namespace-scope
using-declarations sufficed everywhere, and the bulk transform reported
`17 converted, 0 flagged`.

The reason is the one xo-gc's ticket already identified, now confirmed a second
time and for a much larger sample: **the trap is a property of what else the TU
includes, not of the call.** By the time this ticket ran, xo-tokenizer2,
xo-procedure2 and xo-expression2 — all upstream — had been converted, so no
legacy `tag.hpp` reaches a xo-reader2 TU any more and there is no competing
overload to be ambiguous with. The corollary is worth stating: **converting a
subsystem gets cheaper the later it is done**, because each upstream conversion
removes competing overloads from its TUs. Doing this one first would have been
considerably more painful.

### The transform

Scripted, with per-file reporting rather than an assert (the xo-gc lesson) —
`17 converted, 0 flagged`. It reported, per file, the `XO_DEBUG` count and which
using-declarations it inserted, so the output is checkable against the sizing
table above rather than being a bare success message. The three vestigial
includes were removed first, by hand.

Four files got `<xo/ppsink/tag_ostream.hpp>` for their `os << xtag(...)` sites,
each with a comment pointing at the `ostream-containment` milestone.

One follow-up by hand: `utest/SchematikaParser.test.cpp` carried a comment added
during `.xo-backlog/xo-procedure2/issues/02` explaining why its legacy
`scope.hpp` include was explicit ("was arriving via PrimitiveRegistry.hpp").
That comment became false the moment the include became a ppsink one, so it was
deleted. **An explanatory comment attached to a workaround has the same lifetime
as the workaround** — worth checking for whenever one of these tickets closes,
since this series has now added several.

### The `#ifdef NOT_YET` block was left alone, as stated

`DDefineSsm.cpp:64-338` still contains the stale legacy `pretty_print`. The
transform's using-declarations mean its `xtag` would now resolve to `xo::pp` if
the block were ever enabled, but `refrtag` and `ppindentinfo` are still gone, so
it remains uncompilable. Unchanged by design.

## Suggested order

1. delete the three vestigial includes
2. convert the 17 remaining files — bulk transform with **per-file reporting**,
   not an assert (see `.xo-backlog/xo-gc/issues/01`, "A process note"), then
   handle by hand whatever the report flags
3. `ctest` + assertion count, then `nix-build` xo-reader2 and xo-interpreter2
4. drop the three declarations
   (`src/reader2/CMakeLists.txt`, `cmake/xo_reader2Config.cmake.in`,
   `pkgs/xo-reader2.nix`); **re-capture** `subsystem-edges`
   (`.build/reconfigure`, then `--capture-subsystem-edges`, then reinstall
   xo-cmake) and confirm rc=1

`Progress:` falls from 19 to 0 across steps 1 and 2.
