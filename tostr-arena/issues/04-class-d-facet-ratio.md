# 04 — class D: xo-facet and xo-ratio — convert, or make them permanent tostr0

Status: open
Type: decision
Progress: grep -rl '#include <xo/ppsink/tostr_xx\.hpp>' --include=*.hpp --include=*.cpp xo-facet/ xo-ratio/ 2>/dev/null | grep -v '/\.build/' | wc -l

Spec: `.xo-backlog/tostr-arena/spec.md`.

Two subsystems, **one file each**. Neither reaches xo-indentlog2, and neither
declares `xo_ppsink` — so unlike every other class there is no declaration to
swap, only one to write from scratch. That, plus how low they sit, is why this
is a `decision` and not a `refactor`.

```bash
for s in xo-facet xo-ratio; do
  xo-deps --why=$s:xo-indentlog2 -q >/dev/null 2>&1 && echo "$s now reaches il2 -- reclassify"
  grep -rn 'xo_ppsink\|xo_indentlog2' $s/CMakeLists.txt $s/src/*/CMakeLists.txt 2>/dev/null
done
```

## xo-facet is the sharp case

`xo-build --list` puts **xo-facet at position 10 and xo-indentlog2 at 13**, so
converting it moves xo-facet *after* indentlog2 in the level order. No cycle —
indentlog2's upstream closure does not contain xo-facet:

```bash
xo-deps --deps-of=xo-indentlog2 --format=names -q
#   xo-arena xo-indentlog2 xo-ppsink xo-randomgen xo-reflectutil
#   xo-subsys xo-testutil xo-timeutil
```

— but it is a real reordering of the graph's foundations, for this much use:

```bash
grep -n 'tostr0\|tostr_xx' xo-facet/include/xo/facet/FacetRegistry.hpp
#   :17   #include <xo/ppsink/tostr_xx.hpp>
#   :146  using xo::pp::tostr0;
#   :149  throw std::runtime_error(tostr0("FacetRegistry::variant failed", ...
#   :199  using xo::pp::tostr0;
#   :211  << "  [" << tostr0(kv.first.first)
#   :212  << "," << tostr0(kv.first.second) << "]"
```

One error message and one debug dump — **in a public header**, so the new
upstream propagates to every consumer of `FacetRegistry.hpp`:

```bash
xo-deps --users-of=xo-facet --format=names -q
```

## The recommendation, to be argued with rather than followed

**Leave both on `tostr0.hpp` permanently and add them to the reserved list**
(`xo-ppsink/include/xo/ppsink/tostr0.hpp:11-15`). The reasons:

- The reserved list already exists for exactly this — "subsystems that cannot
  use xo-indentlog2 because of leveling topology", plus "a couple of unforced
  choices for principle of least surprise: xo-refcnt xo-flatstring". xo-facet
  and xo-ratio are the same species as those two.
- The benefit is proportional to how much a subsystem renders. Two call sites in
  facet and one in ratio is not a case for pulling in an arena allocator.
- `tostr0` is not deprecated and is not going away; the spec's end state is
  `tostr_xx.hpp` deleted, not `tostr0.hpp` deleted.

The argument the other way: leaving low-level subsystems on a second spelling of
the same function is a permanent split that every future reader has to learn,
and `TempArena` is a thread-local that costs nothing until first use
(`xo-indentlog2/src/indentlog2/TempArena.cpp:19-32`).

**Whichever way this goes, `tostr0.hpp:11-15` must end up describing the tree.**
That header comment is the only place the reserved set is written down, and it
is already one subsystem out of date — `xo-refcnt` is listed as permanent while
`xo-refcnt/include/xo/refcnt/Refcounted.hpp` includes the marker
(`.xo-backlog/tostr-arena/spec.md` notes the same class of drift).

## If the decision is "convert"

Both need a declaration written, not swapped — pick the flavour from what the
subsystem already is:

```bash
grep -rn 'xo_dependency\|xo_headeronly_dependency\|xo_pybind11_dependency' \
  xo-facet/CMakeLists.txt xo-facet/src/*/CMakeLists.txt \
  xo-ratio/CMakeLists.txt xo-ratio/src/*/CMakeLists.txt 2>/dev/null
```

xo-ordinaltree's `xo_headeronly_dependency(${SELF_LIB} xo_indentlog2)` in
`5069be25` is the precedent for a header-only subsystem. Then `pkgs/*.nix`, then
the Config if header-visible, then re-capture `subsystem-edges` — and expect the
build **order** to change, not just the edge set, which is the one diff in this
whole spec that will look alarming and be correct.

## Done when

- a decision is recorded here, with the reasoning, whichever way it goes
- `tostr0.hpp:11-15` describes the tree — including the `xo-refcnt` discrepancy
  above, which should be resolved at the same time since it is the same list
- if converted: `Progress:` returns 0, `nix-build ci.nix -A xo-facet` and
  `-A xo-ratio` green, `subsystem-edges` re-captured
- if not converted: both files move from `tostr_xx.hpp` to `tostr0.hpp`, so
  `Progress:` still returns 0 and the marker header is free to be deleted
