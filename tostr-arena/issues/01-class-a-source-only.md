# 01 — class A: source-only conversions (8 subsystems)

Status: fixed 2026-08-15
Type: refactor
Progress: grep -rlE '\btostr0\b|tostr_xx' --include=*.hpp --include=*.cpp xo-alloc2/ xo-object2/ xo-gc/ xo-numeric/ xo-tokenizer2/ xo-expression2/ xo-reader2/ xo-pyjit/ 2>/dev/null | grep -v '/\.build/' | wc -l

Spec: `.xo-backlog/tostr-arena/spec.md`. Recipe and rationale live there; this
ticket is the scope and the counter.

**These eight already reach xo-indentlog2 and declare no `xo_ppsink` of their
own**, so the conversion is step 1 alone — include swap and call rename. No
CMakeLists change, no `pkgs/*.nix` change, no `subsystem-edges` re-capture.

In build order (`xo-build --list`):

| subsystem | files |
|---|---|
| xo-alloc2 | 1 |
| xo-object2 | 4 |
| xo-gc | 5 |
| xo-numeric | 1 |
| xo-tokenizer2 | 2 |
| xo-expression2 | 1 |
| xo-reader2 | 4 |
| xo-pyjit | 1 |

Re-derive rather than trusting the table (`CONVENTIONS.md` rule 2 — and the
counts move as work lands):

```bash
for s in xo-alloc2 xo-object2 xo-gc xo-numeric xo-tokenizer2 xo-expression2 xo-reader2 xo-pyjit; do
  xo-deps --why=$s:xo-indentlog2 -q >/dev/null 2>&1 || echo "$s NO LONGER class A: does not reach il2"
  grep -rl 'xo_ppsink' $s/CMakeLists.txt $s/src/*/CMakeLists.txt 2>/dev/null \
    && echo "$s NO LONGER class A: declares ppsink"
done
# (no output = the classification still holds)
```

## Do these two first

**xo-object2 (4) and xo-expression2 (1) are half-converted.** `638fd54a` and
`4eb66532` each moved a single file and left the rest, so both currently read as
converted in the commit log while most of their sites are not:

```bash
grep -rl 'ppsink/tostr_xx' --include=*.hpp --include=*.cpp xo-object2/ xo-expression2/ | grep -v '/\.build/'
grep -rl 'indentlog2/print/tostr' --include=*.hpp --include=*.cpp xo-object2/ xo-expression2/ | grep -v '/\.build/'
```

## The one file to look at rather than sed

`xo-alloc/include/xo/alloc/CircularBuffer.hpp` is **not** in this class (xo-alloc
is class B), so every site here is a `.cpp`. Verify that before bulk-editing —
if a public header turns up in this list, the subsystem has moved to class C's
problem and the conversion pushes indentlog2 onto its consumers:

```bash
grep -rl '#include <xo/ppsink/tostr_xx\.hpp>' --include=*.hpp \
  xo-alloc2/ xo-object2/ xo-gc/ xo-numeric/ xo-tokenizer2/ xo-expression2/ xo-reader2/ xo-pyjit/ \
  | grep -v '/\.build/' | grep '/include/'
# (expected: empty)
```

## Verification

Per subsystem:

```bash
xo-build -q --configure --with-utests --with-examples --build --install <sub>
nix-build ci.nix -A <sub> --no-out-link
```

`nix-build` earns its keep even though no nix file changes: it is the only check
that the *installed* package config still resolves, and these subsystems reach
indentlog2 transitively rather than by declaration, which is exactly the shape
that failed for xo-numeric during the earlier indentlog migration.

At the end of the class:

```bash
xo-build --sweep
#   62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped
```

**No `subsystem-edges` re-capture** — nothing here changes a declaration. If a
re-capture *does* show a diff after this ticket, something in the class was
misclassified; go back to the loop above.

## Done when

- `Progress:` returns 0
- the sweep line is unchanged
- one commit per subsystem, matching the existing message shape
  (`xo-<sub>: upgrade to xo-indentlog2 tostr [REFACTOR]`)

## Fixed 2026-08-15 — the conversion was the easy part

All eight converted, 19 files, exactly as scoped: include swap and call rename,
no CMakeLists change, no `pkgs/*.nix` change, no `subsystem-edges` diff. The
class-A prediction held — this is the one class where "source-only" meant it.

`Progress:` has been rewritten to count by **use** rather than by include, to
match what class B had to learn (see `issues/02`). This class genuinely had no
free-riding *code* — but switching the query still found one thing the
include-based version could not: a **stale comment**.

`xo-pyjit/src/pyjit/pyjit.cpp:13` read

```c
/* tostr0/xtag used below; they used to arrive via xo-jit's headers, ... */
```

above code that had been calling `tostr` since the conversion. The comment was
itself introduced during this migration, describing an include that the
migration then changed — the same failure recorded at the end of
`.xo-backlog/xo-reader2/issues/02`: *an explanatory comment attached to a
workaround has the same lifetime as the workaround.* Third instance now.

Worth noting the use-based `Progress:` catches these and the include-based one
cannot, which is a second reason to prefer it beyond the free-rider problem.

xo-object2 and xo-expression2 went first per the note above, so neither is left
reading as converted in the commit log while most of its sites are not.

## But the class exposed three defects that had nothing to do with tostr

None was caused by this ticket; all three were found because it forced a
`nix-build` per subsystem for the first time in a while.

### 1. `pkgs/xo-arena.nix` had no `xo-subsys` — every nix build was broken

`xo-arena/src/arena/CMakeLists.txt:26` gained `xo_dependency(${SELF_LIB} subsys)`
in `fb87ec84` ("xo-indentlog2: toppstr() using TempArena"), and the nix input
never followed. Twelve commits later, **every** `nix-build ci.nix -A <anything>`
failed at xo-arena:

```
CMake Error at .../xo_cxx.cmake:1501 (find_package):
  Could not find a package configuration file provided by "subsys"
Call Stack: ... src/arena/CMakeLists.txt:26 (xo_dependency)
```

Fixed by adding `xo-subsys` to the argument list and `propagatedBuildInputs`.
The umbrella build never saw it because xo-subsys is installed at
`~/local`; only nix isolates hard enough to notice. This is the exact failure
mode `CONVENTIONS.md` gives `nix-build` for, and it had been latent for days
because nobody ran one.

### 2. `xo_pybind11_dependency` does not carry transitive interface include dirs

xo-pyunit and xo-pyreactor were **already failing** at HEAD when this class
started, and xo-pyjit joined them the moment class A converted it. Baseline
measured by stashing the work:

| | HEAD | HEAD + class A |
|---|---|---|
| failing | xo-pyunit, xo-pyreactor | + xo-pyjit |
| skipped | xo-pysimulator, xo-pyprocess, xo-pykalmanfilter | same |

The mechanism is deliberate and documented in the macro itself
(`xo-cmake/cmake/xo_macros/xo_cxx.cmake:1919-1920`):

```cmake
message("-- [${target}] remove ${dep}.INTERFACE_LINK_LIBRARIES to avoid problems with transitive deps")
set_property(TARGET ${dep} PROPERTY INTERFACE_LINK_LIBRARIES "")
```

So a pybind consumer gets only the *directly declared* target's include dirs.
That collided with four includes in xo-indentlog2 that needed a **second**
interface dir to resolve — `print/tostr.hpp` reaching `"LogStreambuf.hpp"`,
`"LogBuffer.hpp"`, `"TempArena.hpp"` one directory up, and `TempArena.hpp`
reaching `"DArena.hpp"` in xo-arena.

xo-pyunit proved a declaration was not enough: `de5e67c6` had already given it
`xo_pybind11_dependency(xo_indentlog2)`, so it got `-isystem .../xo/indentlog2`
and still died on `DArena.hpp`.

**Resolved by RC in xo-indentlog2, not per consumer** — the four includes became
`"../LogStreambuf.hpp"`, `"../LogBuffer.hpp"`, `"../TempArena.hpp"` and
`<xo/arena/DArena.hpp>`. All three pybind modules then built with no extra
declaration anywhere. Worth recording that the alternative on the table —
declaring `xo_indentlog2` **and** `xo_arena` in each pybind consumer — would
have worked too but reopened with every future transitive dependency of
indentlog2.

Note `<xo/arena/DArena.hpp>` was not a deviation at all:
`xo-indentlog2/include/xo/indentlog2/LogBuffer.hpp:9` already spelled it that
way. `TempArena.hpp` was the outlier.

### 3. Converting X can break the nix build of X's *consumers*

`nix-build ci.nix -A xo-expression` passed while `-A xo-pyexpression` failed:

```
Could not find a package configuration file provided by "xo_indentlog2"
  ... xo_expressionConfig.cmake:33 (find_dependency)
```

`xo_dependency` is PUBLIC, so xo-expression's exported Config emits
`find_dependency(xo_indentlog2)` and every nix consumer must supply it — but
`pkgs/xo-expression.nix` had `xo-indentlog2` in `nativeBuildInputs` under
`lib.optionals doCheck`, where it is available for xo-expression's own tests and
propagates to nobody. Moved to `propagatedBuildInputs`; verified by the failing
consumer, not by the subsystem.

**The rule this yields, and which classes B and C then followed:** after
converting a subsystem, `nix-build` a *consumer* of it. `nix-build` on the
subsystem itself cannot see this class of breakage, and neither can the umbrella.

## Notes

**The sweep was already red when this class started, and nobody knew.** Two
pybind modules had been failing at HEAD. The lesson is not about tostr: a class
that begins without establishing the baseline cannot tell its own breakage from
the breakage it inherited. The stash-and-sweep that produced the table above
took one run and should be the first step of any class, not a forensic step
after something looks wrong.
