# 03 — class C: subsystems that gain xo-indentlog2 as a new upstream (5)

Status: open
Type: refactor
Progress: grep -rl '#include <xo/ppsink/tostr_xx\.hpp>' --include=*.hpp --include=*.cpp xo-reflect/ xo-unit/ xo-tokenizer/ xo-webutil/ xo-distribution/ 2>/dev/null | grep -v '/\.build/' | wc -l

Spec: `.xo-backlog/tostr-arena/spec.md`.

**These five do not reach xo-indentlog2 at all today.** Each declares
`xo_ppsink`, so the mechanical edit looks identical to class B — but the effect
is not: the subsystem acquires a new upstream, its `pkgs/*.nix` input becomes
load-bearing rather than redundant, and the level graph grows an edge.

```bash
for s in xo-reflect xo-unit xo-tokenizer xo-webutil xo-distribution; do
  xo-deps --why=$s:xo-indentlog2 -q >/dev/null 2>&1 && echo "$s ALREADY reaches il2 -- reclassify"
done
# (no output = still class C)
```

In build order:

| subsystem | files | public headers among them |
|---|---|---|
| xo-reflect | 3 | `include/xo/reflect/pointer/PointerTdx.hpp` |
| xo-unit | 2 | — |
| xo-tokenizer | 3 | `include/xo/tokenizer/{token,tokenizer}.hpp` |
| xo-webutil | 3 | — |
| xo-distribution | 2 | `include/xo/distribution/{ExplicitDist,KolmogorovSmirnov}.hpp` |

```bash
grep -rl '#include <xo/ppsink/tostr_xx\.hpp>' --include=*.hpp \
  xo-reflect/ xo-unit/ xo-tokenizer/ xo-webutil/ xo-distribution/ \
  | grep -v '/\.build/' | grep '/include/'
```

## Decide before converting, per subsystem

Five of this ticket's thirteen sites are public headers. Converting one of those
does not move a private implementation detail — it puts xo-indentlog2 (and with
it the arena, and `TempArena`'s thread-local scratch) into every consumer's
include graph. Two questions each:

**1. How far does it propagate?**

```bash
xo-deps --users-of=<sub> --format=names -q
```

**2. Is `tostr` load-bearing in that header, or one error message?** If it is one
`throw std::runtime_error(tostr0(...))`, leaving that header on `tostr0.hpp`
while converting the subsystem's `.cpp` files is a legitimate outcome — the two
headers coexist fine in one *subsystem*, just not in one *translation unit*.
That outcome must be recorded in `tostr0.hpp:11-15` alongside the reserved list,
or the next reader will treat it as an oversight.

**xo-reflect and xo-unit are the ones to think hardest about** — both sit low
(build-order 17 and 33) and both are widely depended on. xo-tokenizer,
xo-webutil and xo-distribution sit high enough that the new edge costs little.

## The nix input is real here

Unlike class B, nothing supplies indentlog2 transitively, so a missing
`pkgs/<sub>.nix` entry **will** fail:

```bash
nix-build ci.nix -A <sub> --no-out-link
```

That makes this the one class where the umbrella build and the nix build can
disagree in the direction that gets noticed. Run it per subsystem, not per
class.

## Config.cmake.in

Each of these has a hand-written Config; check whether it names ppsink:

```bash
grep -ln 'xo_ppsink\|XO_FIND_DEPENDENCY_BLOCK' \
  xo-{reflect,unit,tokenizer,webutil,distribution}/cmake/*.cmake.in
```

`xo_dependency` is PUBLIC, so a header-visible dep must appear there
(`xo-cmake/cmake/xo_macros/xo_cxx.cmake:1604-1621`).

## Verification

```bash
xo-build -q --configure --with-utests --with-examples --build --install <sub>
nix-build ci.nix -A <sub> --no-out-link
xo-build --sweep     # 62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped
```

Re-capture `subsystem-edges` after each subsystem in this class rather than once
at the end — these are new edges, not moved ones, and a surprise in the diff is
more informative when it is attributable to one change:

```bash
.build/reconfigure && .build/reconfigure --capture-subsystem-edges
xo-build -q --configure --build --install xo-cmake
git diff xo-cmake/etc/xo/subsystem-edges
```

## Done when

- `Progress:` returns 0, **or** the remainder is public headers deliberately
  left on `tostr0.hpp` — in which case say so here and add them to
  `tostr0.hpp:11-15`, and adjust this ticket's `Progress:` to exclude them so it
  can still reach 0
- each converted subsystem names `xo-indentlog2` in CMakeLists, `pkgs/*.nix`,
  and its Config if header-visible
- `subsystem-edges` re-captured; sweep line unchanged
