# 02 — class B: swap the ppsink declaration (8 subsystems)

Status: fixed 2026-08-15
Type: refactor
Progress: grep -rlE '\btostr0\b|tostr_xx' --include=*.hpp --include=*.cpp xo-alloc/ xo-object/ xo-interpreter/ xo-reactor/ xo-pyreactor/ xo-simulator/ xo-process/ xo-imgui/ 2>/dev/null | grep -v '/\.build/' | grep -vE 'ex3/imgui_ex3\.cpp|ex4a/imgui_ex4a\.cpp' | wc -l

Spec: `.xo-backlog/tostr-arena/spec.md`.

**These eight already reach xo-indentlog2, and each declares `xo_ppsink`
itself.** So the conversion is the full recipe minus the risk: sources, plus
swapping the declaration in CMakeLists and `pkgs/*.nix`. The subsystem gains no
upstream it did not already have — only the *declared* edge moves.

In build order:

| subsystem | files | declaration |
|---|---|---|
| xo-alloc | 7 | `xo-alloc/src/alloc/CMakeLists.txt` |
| xo-object | 2 | `xo-object/src/object/CMakeLists.txt` |
| xo-interpreter | 3 | `xo-interpreter/src/interpreter/CMakeLists.txt` |
| xo-reactor | 1 | `xo-reactor/src/reactor/CMakeLists.txt` |
| xo-pyreactor | 1 | `xo-pyreactor/src/pyreactor/CMakeLists.txt` |
| xo-simulator | 1 | `xo-simulator/src/simulator/CMakeLists.txt` |
| xo-process | 1 | `xo-process/src/process/CMakeLists.txt` |
| xo-imgui | 2 | `xo-imgui/src/imgui/CMakeLists.txt` |

Re-derive before starting:

```bash
for s in xo-alloc xo-object xo-interpreter xo-reactor xo-pyreactor xo-simulator xo-process xo-imgui; do
  xo-deps --why=$s:xo-indentlog2 -q >/dev/null 2>&1 || echo "$s NOT class B: does not reach il2"
  grep -rn 'xo_ppsink' $s/CMakeLists.txt $s/src/*/CMakeLists.txt 2>/dev/null | head -2
done
```

## The public header in this class

`xo-alloc/include/xo/alloc/CircularBuffer.hpp` is the one class-B site that is a
public header. Converting it moves indentlog2 from *declared by xo-alloc* to
*included by everything downstream of xo-alloc*, which is a larger blast radius
than the other seven:

```bash
xo-deps --users-of=xo-alloc --format=names -q
```

Not a blocker — xo-alloc's consumers are already deep in the graph — but it
should be a deliberate conversion, and it is the one file in this ticket worth
reading rather than rewriting mechanically.

## `pkgs/*.nix` is the step that gets forgotten

Three instances so far: xo-pyunit (fixed a commit late in `04f8ae6e`),
xo-ordinaltree (fixed a commit late inside `93f50aa8`), and **xo-jit, still
outstanding**:

```bash
grep -c 'xo-indentlog2' pkgs/xo-jit.nix       # 0
grep -rn 'xo_indentlog2' xo-jit/src/jit/CMakeLists.txt
```

xo-jit builds only because `pkgs/xo-expression.nix` propagates indentlog2. Fold
that fix in here — it is one line, it is the same defect, and xo-jit is adjacent
to this class in build order. After each subsystem:

```bash
nix-build ci.nix -A <sub> --no-out-link
```

Note this will **not** fail for a missing input while the transitive path
survives, exactly as it does not fail for xo-jit today. So also check the
declaration directly rather than inferring it from a green build:

```bash
grep -n 'xo-indentlog2' pkgs/<sub>.nix
```

## Config.cmake.in

`xo_dependency` is `PUBLIC` (`xo-cmake/cmake/xo_macros/xo_cxx.cmake:1604-1621`),
so the dep appears in `INTERFACE_LINK_LIBRARIES` and the exported Config must
resolve it. Where the Config is generated (`@XO_FIND_DEPENDENCY_BLOCK@`) that is
automatic; where it is hand-written and names `xo_ppsink`, swap it:

```bash
grep -ln 'xo_ppsink' xo-{alloc,object,interpreter,reactor,pyreactor,simulator,process,imgui}/cmake/*.cmake.in 2>/dev/null
```

## Verification

Per subsystem, then at the end of the class:

```bash
xo-build -q --configure --with-utests --with-examples --build --install <sub>
nix-build ci.nix -A <sub> --no-out-link
xo-build --sweep     # 62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped
```

**Re-capture `subsystem-edges`** once the class is done — the declared edges
moved:

```bash
.build/reconfigure
.build/reconfigure --capture-subsystem-edges
xo-build -q --configure --build --install xo-cmake
git diff xo-cmake/etc/xo/subsystem-edges
```

The diff should be exactly eight `-> xo-ppsink` edges becoming
`-> xo-indentlog2`, and nothing else. Anything else in it is the finding.

## Done when

- `Progress:` returns 0
- every one of the eight names `xo-indentlog2` in both its CMakeLists and its
  `pkgs/*.nix`, checked by grep and not inferred from a green build
- `pkgs/xo-jit.nix` names it too
- `subsystem-edges` re-captured, diff as described
- sweep line unchanged

## Fixed 2026-08-15 — and three things above were wrong

All eight converted: xo-pyreactor, xo-reactor, xo-interpreter, xo-object,
xo-alloc, xo-simulator, xo-process, xo-imgui. `xo-build --sweep` green
throughout at `62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`, and
`nix-build ci.nix -A <sub>` run per subsystem plus consumers.

### 1. The file counts were include-based, and undercounted by 10

The table above says 18 files. Counted by **use** it was 28. Ten files named
`tostr0` while including no tostr header at all, reaching it through some other
header:

```bash
for s in xo-alloc xo-object xo-interpreter xo-reactor xo-pyreactor xo-simulator xo-process xo-imgui; do
  for f in $(grep -rl '\btostr0\b' --include=*.hpp --include=*.cpp $s/ 2>/dev/null | grep -v '/\.build/'); do
    grep -q 'tostr_xx\.hpp\|tostr0\.hpp' $f || echo "$f"
  done
done
```

This is the same lesson recorded in `.xo-backlog/xo-tokenizer2/issues/01` and
`.xo-backlog/xo-procedure2/issues/02` — **an include census is a lower bound,
never a work-list** — and it was reached for anyway, because the marker header
made an include-based query so convenient. The `Progress:` line has been
rewritten to count by use.

### 2. "CircularBuffer.hpp is the one public header in this class" — wrong; seven

`xo-object/include/xo/object/{Boolean,Float,Integer,String}.hpp` and
`xo-reactor/include/xo/reactor/{Sink,EventStore}.hpp` are the other six, and all
six were free-riders, which is exactly why the include census missed them. The
claim was plausible because it was derived from a query that *did* return real
hits — `CONVENTIONS.md` rule 3's failure mode precisely.

Converting them made indentlog2 header-visible across xo-alloc's 15-subsystem
consumer set and xo-object's 14. It propagated cleanly: no consumer needed a new
declaration, and every consumer nix-built.

### 3. "Swap, do not add" — true once out of eight

Only **xo-pyreactor** was a clean swap. The other seven kept `xo_ppsink`
alongside a new `xo_indentlog2`, because their public headers still name
`xo::pp::{scope,xtag,tag,log_level,PpSink}` directly, which indentlog2
re-exporting ppsink would leave true but undeclared. Each got a comment saying
which of the two the headers need and why:

```bash
grep -rn -B2 'xo_dependency(${SELF_LIB} xo_indentlog2)' \
  xo-{alloc,object,reactor,simulator,process}/src/*/CMakeLists.txt
```

Two of the seven — xo-simulator and xo-process — are the inverse case worth
naming: public headers need **ppsink** (`Simulator.hpp:159` takes an
`xo::pp::scope *`), while indentlog2 is source-only. The declaration cannot show
that distinction, so the comment carries it.

The consequence for the graph: the predicted `subsystem-edges` diff — "exactly
eight `-> xo-ppsink` edges becoming `-> xo-indentlog2`" — was wrong for the same
reason. What actually landed was 10 insertions / 5 deletions, and it includes
edges owed to commits well before this class (`xo-jit`, `xo-kalmanfilter`,
`xo-pyunit`, `xo-ordinaltree`, and `xo-subsys xo-arena` from `fb87ec84`), because
the captured graph had been stale for some time:

```bash
git diff xo-cmake/etc/xo/subsystem-edges
```

`xo-indentlog2 xo-{alloc,object,interpreter}` were already present before this
class, so those three do not appear in the diff despite gaining a declaration.

## Two files deliberately not converted

`xo-imgui/example/{ex3/imgui_ex3.cpp,ex4a/imgui_ex4a.cpp}` hold four
`xo::tostr0(...)` sites naming a symbol that does not exist (`tostr0` is in
`xo::pp`), inside duplicated dead regions that never survive preprocessing. They
were converted once and reverted the same session. They are the subject of
`.xo-backlog/xo-imgui/issues/03-example-dead-duplicates.md`, and are excluded
from this ticket's `Progress:` so it can reach 0.

That ticket also records why the sweep never noticed: `XO_ENABLE_VULKAN` is off
in the working `.build`, so three of the five examples are not compiled by
`--with-examples`.

## `pkgs/xo-jit.nix`

Resolved by `a11b4309 nix build: misc dep fixes` before this class reached it.

## The new shape of the nix slip

The section above warns that a missing `pkgs/<sub>.nix` input can hide behind a
transitive path. Class A found the other direction: **converting X can break the
nix build of X's consumers.** `xo_dependency` is PUBLIC, so X's exported Config
gains `find_dependency(xo_indentlog2)`, and every nix consumer must supply it.
`pkgs/xo-expression.nix` had `xo-indentlog2` under `lib.optionals doCheck` in
`nativeBuildInputs`, so it never propagated, and `nix-build ci.nix -A
xo-pyexpression` failed while `-A xo-expression` passed. Moving it to
`propagatedBuildInputs` fixed it.

So the per-subsystem check is `nix-build` on a **consumer**, not on the
subsystem just converted. That is what the class-B runs did, and it is why
nothing else surfaced.
