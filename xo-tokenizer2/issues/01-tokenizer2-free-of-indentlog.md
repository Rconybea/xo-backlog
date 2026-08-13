# 01 — xo-tokenizer2 free of xo-indentlog

Status: fixed 2026-08-13
Type: refactor
Milestone: ppsink-migration
Progress: grep -rl '#include <xo/indentlog/' xo-tokenizer2/ 2>/dev/null | grep -v '/\.build/' | wc -l

The goal is one command returning **rc=1**:

```bash
xo-deps --why=xo-tokenizer2:xo-indentlog     # today: rc=0, one hop
```

Fourth in the series after xo-object2, xo-gc and xo-procedure2 (all `fixed`).
RC's reason for taking it next (2026-08-13): it **grows the buildable
indentlog-free subgraph** rather than just clearing one subsystem — xo-tokenizer2
sits directly under xo-reader2 and xo-interpreter2, the two subsystems that
still declare xo-indentlog, so it is the next brick that lets a contiguous
region of the graph be built without the legacy library at all.

## What xo-tokenizer2 uses it for

**6 files**, and the shape differs from the earlier three: four of the six are
**public headers**, and one of the two remaining includes is in an example.

```bash
for f in $(grep -rl '#include <xo/indentlog/' xo-tokenizer2/ | grep -v '/\.build/'); do
  printf "%-56s %s\n" "$f" "$(grep -o '<xo/indentlog/[a-z/_0-9]*\.hpp>' $f | tr '\n' ' ')"
done
```

| file | legacy headers | actually uses |
|---|---|---|
| `include/xo/tokenizer2/tokentype.hpp` | `ppdetail_atomic.hpp`, `print/tag.hpp` | **nothing** — see below |
| `include/xo/tokenizer2/Token.hpp` | `print/tag.hpp` | **nothing** |
| `include/xo/tokenizer2/Tokenizer.hpp` | `ppdetail_atomic.hpp`, `scope.hpp` | **nothing** |
| `include/xo/tokenizer2/TokenizerError.hpp` | `scope.hpp` | 1 `scope`, 2 `xtag` |
| `src/tokenizer2/Token.cpp` | `print/tag.hpp` | 12 `tostr`, ~16 `xtag` |
| `example/tokenrepl/tokenrepl.cpp` | `log_config.hpp` | `log_config`, 1 `scope`, 3 `xtag` |

**This table is a lower bound, and turned out to be wrong by two files** — see
"the `Progress:` metric undercounted the work" below. `src/tokenizer2/Tokenizer.cpp`
and `src/tokenizer2/TokenizerError.cpp` use the facility heavily and include
nothing for it, so no include-based census can see them.

So **three of the six headers include the legacy library and use none of it**.
Re-derive before trusting that:

```bash
grep -n 'scope\|\bxtag\b\|\btostr\b\|XO_' xo-tokenizer2/include/xo/tokenizer2/Tokenizer.hpp | grep -v '#include'
# (no output)
```

Those three are pure propagation into xo-reader2 and xo-interpreter2, which is
exactly the thing this ticket exists to stop.

One of them carries a **stale reason**: `tokentype.hpp:9` is annotated
`// for STRINGIFY`, but the file no longer uses it —

```bash
grep -n 'STRINGIFY' xo-tokenizer2/include/xo/tokenizer2/tokentype.hpp
#   :9   #include <xo/indentlog/print/tag.hpp> // for STRINGIFY   <- the comment IS the only hit
```

so the include deletes with the rest. If a *consumer* turns out to have been
living on that propagation, the replacement is `<xo/ppsink/stringify.hpp>`,
which defines the identical macro guarded against redefinition.

## The `PPDETAIL_ATOMIC` blocks are already dead code

`tokentype.hpp:194` and `Token.hpp:231` each carry

```cpp
#ifndef ppdetail_atomic
    namespace print {
        PPDETAIL_ATOMIC(xo::scm::tokentype);
    }
#endif
```

and the guard is **always false**, because the header that would supply
`PPDETAIL_ATOMIC` is the same one that defines the macro away:

```bash
grep -n '#define ppdetail_atomic' xo-indentlog/include/xo/indentlog/print/ppdetail_atomic.hpp
#   :23  #define ppdetail_atomic ppdetail
```

The `#define` is the *debugging switch* described in that header's comment:
suppress it and every participating type must specialize explicitly. In a normal
build it is defined, so both blocks compile to nothing.

Consequence: **include and block delete together, as a no-op.** Not
independently — dropping only the include flips the guard true and the block
then references a `PPDETAIL_ATOMIC` that no longer exists.

There is no ppsink replacement to write. ppsink's `Prettifier<T>` primary
template is empty and falls through to `operator<<` (`pretty_ostream.hpp`,
pulled in by both `scope.hpp` and `tostr.hpp`), which is precisely what
"atomic" meant. Both types already have an `operator<<`
(`tokentype.hpp:186`, `Token.hpp:222`).

## Its own declaration is the only path

```bash
DEPS=$(xo-deps --deps-of=xo-tokenizer2 --format=names -q | tr ' ' '\n' \
       | grep '^xo-' | grep -v '^xo-tokenizer2$' | grep -v '^xo-indentlog$')
for d in $DEPS; do xo-deps --why=$d:xo-indentlog -q >/dev/null 2>&1 && echo "$d reaches xo-indentlog"; done
# (no output)
```

Declared in the usual three places:

- `xo-tokenizer2/src/tokenizer2/CMakeLists.txt:16` — `xo_dependency(${SELF_LIB} indentlog)`
- `xo-tokenizer2/cmake/xo_tokenizer2Config.cmake.in:11` — `find_dependency(indentlog)`
- `pkgs/xo-tokenizer2.nix:7,34` — argument + `propagatedBuildInputs`

xo-ppsink is already reachable (`xo-deps --deps-of=xo-tokenizer2` lists it, via
xo-stringtable2), so nothing new is declared — same as xo-gc and xo-procedure2.

## The conversion

| from | to |
|---|---|
| `#include <xo/indentlog/scope.hpp>` | `<xo/ppsink/scope.hpp>` + `<xo/ppsink/scope_macros.hpp>` |
| `#include <xo/indentlog/print/tag.hpp>` | `<xo/ppsink/tag.hpp>` |
| `tostr(...)` | also `<xo/ppsink/tostr.hpp>` |
| `os << xtag(...)` | also `<xo/ppsink/tag_ostream.hpp>` |
| `scope log(XO_DEBUG(f))` | `scope log(XO_DEBUG_(f))` |
| `xo::log_config::min_log_level` | `xo::pp::scope_config::min_log_level` |
| `xo::log_level::severe` | `xo::pp::log_level::severe` |
| `xo::scope` / `xo::xtag` / `xo::tostr` | `xo::pp::` equivalents |

`TokenizerError.hpp` is a header, so its `using` goes **inside the function
body** (house rule), not at namespace scope. `Token.cpp` takes namespace-scope
using-declarations per `xo-arena/src/arena/mmap_util.cpp:16-20`.

`Token::print(std::ostream &)` at `Token.cpp:249` does `os << xtag(...)`, so
that file needs the `tag_ostream.hpp` bridge — same as the five xo-gc utest
files that used `INFO(tostr(xtag(...)))`. Rewriting `print()` onto a `PpSink &`
belongs to the `ostream-containment` milestone, not here.

### Two things to check rather than assume

**`tostr` renders the same, `tosn` does not.** The xo-procedure2 conversion
found that legacy `tosn` appends a newline where `xo::pp::tostr` does not
(recorded in `.xo-backlog/xo-procedure2/issues/02`). `Token.cpp` uses plain
`tostr`, which should be byte-identical — but these strings become
`std::runtime_error` messages, so **measure before/after** rather than assume.
The cheap check is the scratch-program comparison used for `tosn`.

**Nothing pins the token rendering locally.** xo-tokenizer2 has **no `utest/`
directory at all** — it is one of the 28 "no tests" subsystems in
`xo-build --sweep`:

```bash
ls -d xo-tokenizer2/utest        # No such file or directory
```

So the safety net is entirely downstream: xo-reader2's parser suites exercise
the tokenizer heavily, and they are what will notice a rendering change. Worth
stating plainly, because "sweep is green" means much less here than it did for
xo-gc (8578 assertions) — a test gap that predates this ticket, and might
deserve one of its own.

## Expect downstream exposure — more than xo-procedure2 caused

Three public headers stop propagating `scope.hpp` / `tag.hpp` at once, into two
subsystems that use those names heavily. Upper bound on the files that could be
affected — an over-count, since the pattern also matches files already on
`xo::pp::`:

```bash
for s in xo-reader2 xo-interpreter2; do
  for f in $(grep -rlE '\b(xtag|tostr|scope)\b' --include=*.hpp --include=*.cpp $s/ | grep -v '/\.build/'); do
    grep -q '#include <xo/indentlog/' $f || echo "$f"
  done
done
#   xo-reader2: ParserResult.cpp, ParserStack.cpp, ParserStateMachine.hpp,
#               parser/DSchematikaParser.hpp, printable_render.test.cpp,
#               example/readerreplxx/readerreplxx.cpp
#   xo-interpreter2: printable_render.test.cpp, vsm/DVirtualSchematikaMachine.hpp
```

The compiler is the arbiter. Precedent from xo-procedure2: give each exposed
file an explicit `#include <xo/indentlog/scope.hpp>` (or `print/tag.hpp`) with a
comment saying where it used to come from — both subsystems still **declare**
xo-indentlog, so converting them belongs to their own tickets, not this one.

## Verification

`xo-build --sweep` cannot see a dropped declaration; `nix-build` is the only
real check, and here it matters twice over because the change removes headers
from the public surface:

```bash
nix-build ci.nix -A xo-tokenizer2 --no-out-link
xo-deps --users-of=xo-tokenizer2 --format=names -q     # xo-interpreter2 xo-reader2
```

Build both consumers under nix, not just locally. And **build with
`--with-examples`** — `example/tokenrepl/tokenrepl.cpp` is one of the six files,
and it is invisible to a build that omits examples.

## Fixed 2026-08-13

```bash
xo-deps --why=xo-tokenizer2:xo-indentlog     # rc=1
```

`Progress:` 6 -> 0, `subsystem-edges` re-captured (diff was exactly
`xo-indentlog xo-tokenizer2`), `xo-build --sweep` →
`62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`, `nix-build` green
for xo-tokenizer2 and both consumers. xo-reader2's suite — the real safety net,
since xo-tokenizer2 has none — is unchanged at **1837 assertions in 33 test
cases**.

### The goal metric

RC's reason for taking this subsystem was to grow the buildable indentlog-free
region. Measured after:

```bash
for s in $(xo-build --list | tr ' ' '\n' | grep '^xo-' | sort -u); do
  [ -f "$s/CMakeLists.txt" ] || continue; [ "$s" = xo-indentlog ] && continue
  xo-deps --why=$s:xo-indentlog -q >/dev/null 2>&1 && echo "$s"
done
#   xo-expression2
#   xo-interpreter2
#   xo-reader2
```

**Three** subsystems still reach xo-indentlog, out of 62. Every one of them
declares it directly — there is no remaining transitive path to cut, so the
legacy library is now reachable only from the v2 front-end cluster.

Sizes for whoever takes them next (`grep -rl '#include <xo/indentlog/'`, minus
`.build`): xo-expression2 **5** files, xo-interpreter2 **8**, xo-reader2 **19**.

### The `Progress:` metric undercounted the work — twice

The ticket sized this at 6 files, from `grep -rl '#include <xo/indentlog/'`.
The real count was **8**, and both extras were found by the compiler:

- `src/tokenizer2/Tokenizer.cpp` — 3 `scope`, 29 `xtag`, 2 `tostr`, and a
  `log_config::style` assignment. **No legacy include of its own**; it lived on
  `Tokenizer.hpp`.
- `src/tokenizer2/TokenizerError.cpp` — 5 `xtag`, all in `os << xtag(...)` form.
  Same story via `TokenizerError.hpp`.

This is the xo-procedure2 lesson recurring one level down. There it was a whole
subsystem (xo-numeric) invisible to the census; here it is two files invisible
*inside the subsystem being converted*. **A file that uses a facility need not
include anything at all**, so no include-based census can be complete — the
count is a lower bound and should be read as one. The reliable enumeration is by
*use*:

```bash
for f in $(grep -rlE '\b(xtag|tostr|scope)\b' --include=*.hpp --include=*.cpp xo-tokenizer2/ | grep -v '/\.build/'); do
  grep -q 'xo::pp::\|<xo/ppsink/' $f || echo "$f"
done
```

which reaches 0 at the same time as `Progress:`, and unlike it, cannot miss a
file that used the facility without including it.

A third instance of the same shape, in a consumer: xo-interpreter2's
`utest/VirtualSchematikaMachine.test.cpp` **does** include legacy indentlog —
just `print/hex.hpp`, not `scope.hpp`. It got `scope`/`xtag`/`XO_DEBUG` from
Tokenizer.hpp. So "has a legacy include" is no better a proxy than "has none":
only the symbol matters, not the header.

### `STRINGIFY` was a live propagation after all

`tokentype.hpp:9` carried `// for STRINGIFY` and the file itself did not use it
— which is what this ticket recorded before the edit. But `tokentype.cpp` did,
through its `CASE(x)` macro, and reached it via `tokentype.hpp`. The comment was
attached to the wrong file, not stale. Fixed with an explicit
`#include <xo/ppsink/stringify.hpp>` in `tokentype.cpp`, which is the same macro
guarded against redefinition.

### Two consumers exposed, both dead `ppdetail_atomic` blocks

`xo-reader2/include/xo/reader2/DProgressSsm.hpp:246` and
`xo-reader2/include/xo/reader2/apply/DApplySsm.hpp:228` carry the same
`#ifndef ppdetail_atomic` blocks, and were relying on **xo-tokenizer2** to
propagate the header that defines the macro away. Deleted rather than given a
legacy include: they were unreachable code by construction, and deleting them
is what `xo-reactor/include/xo/reactor/PollingReactor.hpp:53` and
`xo-unit/include/xo/unit/quantity_iostream.hpp:28` already did — both carry a
comment beginning "It was dead code. It sat under `#ifndef ppdetail_atomic`".
So this is established practice in the tree, not a new judgement.

The remaining `#ifndef ppdetail_atomic` guards are inside xo-indentlog itself
(where they are the switch mechanism) plus `xo-refcnt/include/xo/refcnt/pretty_refcnt.hpp`:

```bash
grep -rn '#ifndef ppdetail_atomic' --include=*.hpp --include=*.cpp xo-*/ | grep -v '/\.build/'
```

### Rendering measured, not assumed

The ticket asked for `tostr` to be checked rather than trusted, after the `tosn`
newline surprise in xo-procedure2. A scratch program compared legacy vs ppsink
`tostr` across the three argument shapes `Token.cpp` actually uses — an enum
rendered by its own `operator<<`, a `std::string`, and a `char`. **All three
byte-identical**, colour escapes included. So the `std::runtime_error` messages
are unchanged.

`Tokenizer.cpp:134` sets a process-wide style from inside `assemble_token`:

```cpp
xo::pp::scope_config::function_style = xo::FunctionStyle::streamlined;
```

Preserved verbatim (ppsink's default is already `streamlined`, so it is now a
no-op), but a per-call write to a global config is odd enough to flag — it
predates this work and was not touched.

## Suggested order

1. the three unused-include headers (`tokentype.hpp`, `Token.hpp`,
   `Tokenizer.hpp`) — delete include **and** matching `#ifndef ppdetail_atomic`
   block together; this is what actually stops the propagation
2. build xo-reader2 + xo-interpreter2 locally and fix the exposed files with
   explicit includes; then `nix-build` both
3. `TokenizerError.hpp` and `Token.cpp` — the real conversions
4. `example/tokenrepl/tokenrepl.cpp`
5. drop the three declarations; **re-capture** `subsystem-edges` (generated, not
   authored — `.build/reconfigure` then `.build/reconfigure
   --capture-subsystem-edges`, then reinstall xo-cmake so `xo-deps` reads it)
   and confirm `xo-deps --why=xo-tokenizer2:xo-indentlog` returns rc=1

`Progress:` counts files still including legacy indentlog anywhere under
`xo-tokenizer2/`, so it falls from 6 to 0 across steps 1, 3 and 4.
