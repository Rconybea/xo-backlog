# 02 — class B: swap the ppsink declaration (8 subsystems)

Status: open
Type: refactor
Progress: grep -rl '#include <xo/ppsink/tostr_xx\.hpp>' --include=*.hpp --include=*.cpp xo-alloc/ xo-object/ xo-interpreter/ xo-reactor/ xo-pyreactor/ xo-simulator/ xo-process/ xo-imgui/ 2>/dev/null | grep -v '/\.build/' | wc -l

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
