# 02 — xo-interpreter2 free of xo-indentlog (the last one)

Status: fixed 2026-08-13
Type: refactor
Milestone: ppsink-migration
Progress: grep -rl '#include <xo/indentlog/' xo-interpreter2/ 2>/dev/null | grep -v '/\.build/' | wc -l

The goal is one command returning **rc=1**:

```bash
xo-deps --why=xo-interpreter2:xo-indentlog     # today: rc=0
```

and, uniquely for this one, a second command going empty:

```bash
xo-deps --users-of=xo-indentlog --format=names -q
#   today:  xo-indentlog xo-interpreter2
#   after:  xo-indentlog
```

**This is the last subsystem in the tree that reaches xo-indentlog.** Sixth and
final in the series after xo-object2, xo-gc, xo-procedure2, xo-tokenizer2,
xo-expression2 and xo-reader2:

```bash
for s in $(xo-build --list | tr ' ' '\n' | grep '^xo-' | sort -u); do
  [ -f "$s/CMakeLists.txt" ] || continue; [ "$s" = xo-indentlog ] && continue
  xo-deps --why=$s:xo-indentlog -q >/dev/null 2>&1 && echo "$s"
done
#   xo-interpreter2
```

## The easiest of the six

**8 files**, and every one of them uses only `scope` + `xtag`. No `tostr`, no
`os << xtag(...)`, no `quot`/`cond`/`concat`, no `ppdetail_atomic` blocks, no
public header involved — all eight are `.cpp`.

```bash
for f in $(grep -rl '#include <xo/indentlog/' xo-interpreter2/ | grep -v '/\.build/' | sort); do
  printf "%-52s %s\n" "${f#xo-interpreter2/}" "$(grep -o '<xo/indentlog/[a-z/_0-9]*\.hpp>' $f | tr '\n' ' ')"
done
```

| file | `scope` | `xtag` |
|---|---|---|
| `utest/VirtualSchematikaMachine.test.cpp` | 4 | 14 |
| `src/interpreter2/SetupInterpreter2.cpp` | 3 | 11 |
| `src/interpreter2/DVirtualSchematikaMachine.cpp` | 3 | 8 |
| `src/interpreter2/VsmPrimitives.cpp` | 1 | 5 |
| `src/interpreter2/DClosure.cpp` | 1 | 1 |
| `src/interpreter2/DLocalEnv.cpp` | 1 | 0 |
| `src/interpreter2/init_interpreter2.cpp` | 1 | 0 |
| `src/skrepl/skreplxx.cpp` | 1 | 0 |
| **total** | **15** | **39** |

An eighth of xo-reader2's 125 scopes.

### Two details

**`utest/VirtualSchematikaMachine.test.cpp:20` includes
`<xo/indentlog/print/hex.hpp>` and never calls `hex_view`** — vestigial, deletes
outright. (ppsink has `hex.hpp` with `hex_view(range, style)` if a use ever
appears; nothing needs it now.)

**Its `scope.hpp` include was added by hand during
`.xo-backlog/xo-tokenizer2/issues/01`**, when that conversion stopped
`Tokenizer.hpp` propagating the legacy header, together with a comment saying
so. Both the include and **the comment** go — see the note at the end of
`.xo-backlog/xo-reader2/issues/02`: an explanatory comment attached to a
workaround has the same lifetime as the workaround, and this series has left
several around.

`src/skrepl/skreplxx.cpp` is an executable whose own
`src/skrepl/CMakeLists.txt:8` already declares `xo_indentlog2`; it was reaching
legacy `scope.hpp` through the library's propagated dependency.

### The census is clean this time — verified by use, not by include

```bash
for f in $(grep -rlE '\b(xtag|tostr|scope|refrtag|tosn|quot|hex|concat|unq|pad)\b' \
           --include=*.hpp --include=*.cpp xo-interpreter2/ | grep -v '/\.build/'); do
  grep -q '#include <xo/indentlog/' $f || echo "$f"
done
#   include/xo/interpreter2/vsm/DVirtualSchematikaMachine.hpp   (comment only)
#   utest/printable_render.test.cpp                             (already xo::pp::xtag)
```

Both already accounted for, so 8 is exact. Worth running anyway: this check
found two missed files in xo-tokenizer2 and a whole missed subsystem
(xo-numeric) in xo-procedure2.

## Nothing upstream, nothing downstream

```bash
DEPS=$(xo-deps --deps-of=xo-interpreter2 --format=names -q | tr ' ' '\n' \
       | grep '^xo-' | grep -v '^xo-interpreter2$' | grep -v '^xo-indentlog$')
for d in $DEPS; do xo-deps --why=$d:xo-indentlog -q >/dev/null 2>&1 && echo "$d reaches xo-indentlog"; done
# (no output -- xo-reader2 and xo-expression2 were the last, both closed 2026-08-13)

xo-deps --users-of=xo-interpreter2 --format=names -q
# (empty -- it is a leaf)
```

So its own declaration is the only path **and** there is no consumer to expose.
That makes this the one conversion in the series with no `nix-build`-only
failure mode from a dropped propagated dependency — but still run `nix-build`,
because that is also the check on the installed package config.

Declared in three places:

- `xo-interpreter2/src/interpreter2/CMakeLists.txt:57`
- `xo-interpreter2/cmake/xo_interpreter2Config.cmake.in:11`
- `pkgs/xo-interpreter2.nix:10,38` — note this one declares **both**
  `xo-indentlog` and `xo-indentlog2`; only the former goes.

## The conversion

The usual table, minus everything this subsystem does not use:

| from | to |
|---|---|
| `#include <xo/indentlog/scope.hpp>` | `<xo/ppsink/scope.hpp>` + `<xo/ppsink/scope_macros.hpp>` |
| `scope log(XO_DEBUG(f))` | `scope log(XO_DEBUG_(f))` |
| `xo::scope` / `xo::xtag` | `xo::pp::scope` / `xo::pp::xtag` |

ADL should be quiet, for the reason confirmed twice now (xo-gc, xo-reader2):
after the six upstream conversions no legacy `tag.hpp` reaches these TUs, so
there is no competing overload. Namespace-scope using-declarations per
`xo-arena/src/arena/mmap_util.cpp:16-20`.

## Verification

```bash
ctest --test-dir xo-interpreter2/.build
./xo-interpreter2/.build/utest/utest.interpreter2 | tail -3
#   177 assertions in 22 test cases   (measured 2026-08-13, before this ticket)
nix-build ci.nix -A xo-interpreter2 --no-out-link
```

Build with `--with-examples`: `src/skrepl/skreplxx.cpp` is one of the eight.

## What closing this ticket means

`xo-deps --users-of=xo-indentlog` goes empty. xo-indentlog is still **built**
(it is in `xo-build --list` and has its own `utest/`), but nothing in the tree
depends on it any more.

That raises a question this ticket does **not** settle: whether xo-indentlog
should be retired, frozen, or kept as a reference implementation. It is a
separate decision with its own consequences (it still owns the pretty-printer
design the ppsink stack descends from), and it wants its own ticket rather than
being smuggled in here.

## Fixed 2026-08-13 — and with it, the whole series

```bash
xo-deps --why=xo-interpreter2:xo-indentlog     # rc=1
xo-deps --users-of=xo-indentlog --format=names -q
# (empty)

for s in $(xo-build --list | tr ' ' '\n' | grep '^xo-' | sort -u); do
  [ -f "$s/CMakeLists.txt" ] || continue; [ "$s" = xo-indentlog ] && continue
  xo-deps --why=$s:xo-indentlog -q >/dev/null 2>&1 && echo "$s"
done
# (empty)
```

**No subsystem in the tree depends on xo-indentlog.**

`Progress:` 8 -> 0, `subsystem-edges` re-captured (diff exactly
`xo-indentlog xo-interpreter2`), `xo-build --sweep` →
`62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`, `nix-build` green.
The suite is unchanged at **177 assertions in 22 test cases**.

Exactly as predicted, and the least eventful of the six: 8 converted, 0 flagged,
no ADL, no consumer exposed (it is a leaf), no rendering question. The
`print/hex.hpp` include and the workaround comment both deleted as prescribed.

## Three files still include xo-indentlog, and none of them is built

```bash
grep -rl '#include <xo/indentlog/' --include=*.hpp --include=*.cpp xo-*/ \
  | grep -v '/\.build/' | grep -v '^xo-indentlog/'
#   xo-object2/utest/X1Collector.test.cpp
#   xo-object2/utest/Printable.test.cpp
#   xo-refcnt/include/xo/refcnt/Refcounted_indentlog.hpp
```

This looks alarming next to the empty `--users-of` above, so it is worth being
precise about why both are true.

**The two xo-object2 test files are not compiled.** They are commented out of
their own build, with a note saying where they went:

```bash
grep -n 'test.cpp' xo-object2/utest/CMakeLists.txt
#   :6      DArray.test.cpp
#   :7      printable_render.test.cpp
#   :10  #  X1Collector.test.cpp   # moved to xo-gc/
#   :11  #  Printable.test.cpp     # moved to xo-gc/
```

`xo-gc/utest/X1Collector.test.cpp` exists and is **510 lines against the
orphan's 283**, so the live copy has moved on; the xo-object2 copy is a stale
duplicate, not a backup. There is no `Printable.test.cpp` in xo-gc at all — that
one's coverage went somewhere else or was dropped, which is worth checking
before deleting it.

**`Refcounted_indentlog.hpp` is deliberate and unreferenced.** Nothing in the
tree includes it:

```bash
grep -rln 'Refcounted_indentlog' --include=*.hpp --include=*.cpp xo-*/ | grep -v '/\.build/'
#   xo-refcnt/include/xo/refcnt/Refcounted_indentlog.hpp     (itself only)
```

It is an opt-in adapter for a consumer that has legacy indentlog — the same
shape as `xo-ppsink/include/xo/ppsink/*_ostream.hpp`. It costs nothing while
unused, and removing it is a decision about xo-refcnt's public surface, not part
of this ticket.

So the grep is a **file-level** count and the `--why` queries are
**dependency-level**; here they disagree only because dead files are not
dependencies. Left as-is rather than tidied in passing: deleting source files is
the repo owner's call, and the `Printable.test.cpp` coverage question above
needs an answer first.

## What is now open

`xo-indentlog` is still built (it is in `xo-build --list`, has its own `utest/`,
and `pkgs/xo-indentlog.nix` is still referenced by `xo-printjson`, `xo-reactor`,
`xo-callback`, `xo-webutil`, `xo-userenv` and `xo-userenv-slow`), but no
subsystem's cmake declares it.

```bash
grep -ln 'xo-indentlog[^2]' pkgs/*.nix
```

Note the mismatch: those six nix packages still pass `xo-indentlog` even though
the corresponding cmake no longer asks for it. Harmless — an unused
`propagatedBuildInput` — but it is now the only place the dependency survives,
and cleaning it up is the natural next step. Whether xo-indentlog should then be
retired, frozen, or kept as the reference implementation of the pretty-printer
design the ppsink stack descends from is a separate decision, and wants its own
ticket.

## Suggested order

1. delete the vestigial `print/hex.hpp` include, and the stale explanatory
   comment above the `scope.hpp` include in the same file
2. convert the 8 files — bulk transform with **per-file reporting**, not an
   assert
3. `ctest` + assertion count; `nix-build`
4. drop the three declarations; **re-capture** `subsystem-edges`
   (`.build/reconfigure`, then `--capture-subsystem-edges`, then reinstall
   xo-cmake) and confirm both commands at the top of this ticket

`Progress:` falls from 8 to 0 across steps 1 and 2.
