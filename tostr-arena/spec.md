# tostr-arena — move xo off `tostr0()` onto xo-indentlog2's arena-backed `tostr()`

Status: open
Type: spec
Progress: grep -rl '#include <xo/ppsink/tostr_xx\.hpp>' --include=*.hpp --include=*.cpp . 2>/dev/null | grep -v '/\.build/' | wc -l

`xo-indentlog2` gained an arena-backed `xo::pp::tostr()`
(`xo-indentlog2/include/xo/indentlog2/print/tostr.hpp`) that renders through a
`FlatSink` over a thread-local `TempArena` instead of a `std::stringstream`.
It is byte-identical to the old one — measured 2026-08-15, same arguments
through both, `od -c` diff clean, colour escapes included.

Because both spellings live in `xo::pp` with the same signature, they cannot
coexist in one translation unit (`error: redefinition of 'template<class ... Ts>
std::string xo::pp::tostr(const Ts& ...)'`). So the migration was staged rather
than flag-dayed:

- **`xo/ppsink/tostr0.hpp`** — the FlatSink-over-stringstream one, renamed
  `tostr0()`. **Permanent** for subsystems that sit below xo-indentlog2 in the
  level order, listed in that header at `:11-15`.
- **`xo/ppsink/tostr_xx.hpp`** — a marker that defines nothing and just forwards
  to `tostr0.hpp`. Including it means *"this subsystem has not been assigned
  yet."* Call sites already say `tostr0(...)`; only the include distinguishes
  the two camps, which is what makes the intermediate set greppable.

This spec is the walk from `tostr_xx.hpp` to zero. `tostr_xx.hpp` is deleted
when it gets there.

## The recipe

Derived from the six worked examples, of which `16023180` (xo-kalmanfilter) is
the most complete:

```bash
git log --oneline -8 --name-only
#   4eb66532 xo-expression2  93f50aa8 xo-expression  5069be25 xo-ordinaltree
#   1b41f213 xo-reader       638fd54a xo-object2     16023180 xo-kalmanfilter
#   de5e67c6 xo-pyunit       04f8ae6e xo-pyunit (nix follow-up)
```

1. **Sources** — `<xo/ppsink/tostr_xx.hpp>` → `<xo/indentlog2/print/tostr.hpp>`;
   `tostr0(` → `tostr(`; `using xo::pp::tostr0;` → `using xo::pp::tostr;`
2. **`CMakeLists.txt`** — *swap*, do not add:
   `xo_dependency(${SELF_LIB} xo_ppsink)` → `xo_indentlog2`. ppsink still
   arrives, because indentlog2 exports it:
   ```bash
   grep -n INTERFACE_LINK_LIBRARIES ~/local/lib/cmake/xo_indentlog2/xo_indentlog2Targets.cmake
   #   "xo_arena;xo_ppsink"
   ```
   Match the macro flavour already in use — `xo_headeronly_dependency`
   (xo-ordinaltree), `xo_pybind11_dependency` (xo-pyunit).
3. **`pkgs/<sub>.nix`** — the argument list *and* `propagatedBuildInputs`.
4. **`cmake/xo_<sub>Config.cmake.in`** — only if it names `xo_ppsink` by hand.
   Where it is `@XO_FIND_DEPENDENCY_BLOCK@` (xo-kalmanfilter) it regenerates;
   none of the six needed an edit. `xo_dependency` is `PUBLIC`
   (`xo-cmake/cmake/xo_macros/xo_cxx.cmake:1604-1621`), so a hand-written Config
   that named ppsink does need the swap.

**Step 3 is the one that slips.** It was a separate follow-up commit twice —
`04f8ae6e` for xo-pyunit, and xo-ordinaltree's nix change landed a commit late
inside `93f50aa8`. The umbrella build cannot see it; only
`nix-build ci.nix -A <sub> --no-out-link` can.

## The four classes

Measured 2026-08-15. **23 subsystems, 52 files.** None is inside indentlog2's
upstream closure, so all 23 are eligible:

```bash
xo-deps --deps-of=xo-indentlog2 --format=names -q
#   xo-arena xo-indentlog2 xo-ppsink xo-randomgen xo-reflectutil
#   xo-subsys xo-testutil xo-timeutil
```

Membership was derived by two questions per subsystem — *does it already reach
xo-indentlog2?* and *does it declare xo_ppsink itself?* — which is what decides
how much wiring the conversion needs:

```bash
SUBS=$(grep -rl '#include <xo/ppsink/tostr_xx\.hpp>' --include=*.hpp --include=*.cpp . \
       | grep -v '/\.build/' | sed 's|^\./||' | cut -d/ -f1 | sort -u)
for s in $SUBS; do
  xo-deps --why=$s:xo-indentlog2 -q >/dev/null 2>&1 && r=reaches || r=NEW
  d=$(grep -rl 'xo_ppsink' $s/CMakeLists.txt $s/src/*/CMakeLists.txt 2>/dev/null | head -1)
  printf "%-16s %-8s %s\n" "$s" "$r" "${d:-no-ppsink-decl}"
done
```

| class | reaches il2 | declares ppsink | what it costs | ticket |
|---|---|---|---|---|
| **A** | yes | no | source edit only | `issues/01` |
| **B** | yes | yes | + declaration swap, + nix | `issues/02` |
| **C** | **no** | yes | + a genuinely new upstream edge | `issues/03` |
| **D** | **no** | no | + a declaration written from scratch | `issues/04` |

Re-derive membership with the loop above rather than trusting the split if the
graph has moved since — `CONVENTIONS.md` rule 2.

## Sequencing

**A and B first, in build order** — 16 subsystems, 37 files, all mechanical and
each independently verifiable. **Then C and D one at a time**, because each is a
design call rather than a rename (see below). Build order is `xo-build --list`;
it is what makes a single subsystem's conversion checkable in isolation.

## Why C and D are not mechanical

Seven of the 52 sites are **public headers**, and they cluster almost exactly on
C and D:

```bash
grep -rl '#include <xo/ppsink/tostr_xx\.hpp>' --include=*.hpp . \
  | grep -v '/\.build/' | sed 's|^\./||' | grep '/include/'
#   xo-facet/include/xo/facet/FacetRegistry.hpp
#   xo-reflect/include/xo/reflect/pointer/PointerTdx.hpp
#   xo-tokenizer/include/xo/tokenizer/{token,tokenizer}.hpp
#   xo-distribution/include/xo/distribution/{ExplicitDist,KolmogorovSmirnov}.hpp
#   xo-alloc/include/xo/alloc/CircularBuffer.hpp
```

For those subsystems the conversion does not swap a private implementation
detail — it pushes the arena-backed logging stack onto every consumer of a
low-level header. That is a decision each one owes an argument for. `issues/04`
carries the sharpest case.

## Verification

Per subsystem, before the next one starts:

```bash
xo-build -q --configure --with-utests --with-examples --build --install <sub>
nix-build ci.nix -A <sub> --no-out-link     # the ONLY check on step 3
```

At the end of each class:

```bash
xo-build --sweep
#   62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped
```

For B, C and D the **declared** edge moves, so `subsystem-edges` must be
re-captured (`.build/reconfigure`, then `.build/reconfigure
--capture-subsystem-edges`, then `xo-build -q --configure --build --install
xo-cmake`) and the diff checked to be only the expected edge. Class A changes no
declaration and needs no re-capture.

## Done when

- the `Progress:` command above returns 0
- `xo-ppsink/include/xo/ppsink/tostr_xx.hpp` is deleted
- `tostr0.hpp:11-15`'s reserved list matches the tree, i.e. every remaining
  `tostr0.hpp` includer is a subsystem that list names — plus whichever of
  xo-facet / xo-ratio the class-D ticket decides to leave behind
- `xo-build --sweep` green, and `nix-build ci.nix -A xo-userenv --no-out-link`

## Two defects found while measuring (2026-08-15)

**1. `pkgs/xo-jit.nix` declares `xo-ppsink` but not `xo-indentlog2`,** although
xo-jit's sources are already converted and its CMakeLists declares
`xo_indentlog2`:

```bash
grep -c 'xo-indentlog2' pkgs/xo-jit.nix      # 0
grep -rc 'xo_indentlog2' xo-jit/src/jit/CMakeLists.txt
```

It builds only because `pkgs/xo-expression.nix` propagates indentlog2, so the
dependency is real but implicit and breaks the moment xo-expression drops it.
Same slip as xo-ordinaltree and xo-pyunit — third instance, and the first that
did *not* get a follow-up commit because nothing failed. Fix it with whichever
class-B subsystem is convenient, or split it out.

**2. xo-object2 and xo-expression2 are half-converted.** `638fd54a` and
`4eb66532` each moved one file and left the rest:

```bash
grep -rl 'ppsink/tostr_xx' --include=*.hpp --include=*.cpp xo-object2/ xo-expression2/ \
  | grep -v '/\.build/'
```

Both are class A, so finishing them is cheap — worth doing early so they stop
reading as done in the commit log.
