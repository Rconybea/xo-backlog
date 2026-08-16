# 03 — class C: subsystems that gain xo-indentlog2 as a new upstream (5)

Status: fixed 2026-08-16
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

## Fixed 2026-08-16

All five converted: xo-reflect, xo-webutil, xo-distribution, xo-unit,
xo-tokenizer. 13 files. Sweep green throughout at
`62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`.

### The one class where the include census was right

Counted by use, class C was 13 files — the same as by include. No free-riders,
unlike class B's 18-vs-28. Worth recording as the exception rather than letting
"always undercounts" become the new carried label: it undercounts when a
subsystem has headers that reach `tostr0` through *other* headers, and class C's
members mostly did not.

### "Swap, do not add" held zero times out of five

Every one kept `xo_ppsink` and gained `xo_indentlog2` alongside it, because
every one names `xo::pp` outside `tostr` — `PpSink`, `Prettifier`, `scope`,
`xtag`, `quot`, `unq`, `STRINGIFY`. Two shapes are worth distinguishing, and
neither is visible from the declaration, so each carries a comment:

- **ppsink public, indentlog2 source-only** — xo-webutil (`StreamEndpointDescr.hpp`
  names `PpSink`; only the `.cpp`s call `tostr`)
- **both public** — xo-reflect, xo-distribution, xo-tokenizer

### xo-reflect: 34 consumers, and nothing downstream needed a change

RC chose to convert `PointerTdx.hpp` rather than keep it on `tostr0`, reasoning
that xo-reflect2 will be arena-based and unlocks indentlog2 anyway. The
resulting `subsystem-edges` diff was a single line, `+xo-indentlog2 xo-reflect`,
and every consumer built. That is the fourth consecutive class in which making
indentlog2 header-visible propagated with no downstream edit; it should stop
being treated as the risky part.

Its Config is **hand-written**, so `find_dependency(xo_indentlog2)` had to be
added by hand — `xo-unit` is the only other one like that; the remaining three
are `@XO_FIND_DEPENDENCY_BLOCK@`.

### xo-unit reclassified mid-flight — and the reclassification was wrong

The guard at the top of this ticket reported that xo-unit had come to *reach*
xo-indentlog2, via `xo-unit -> xo-ratio -> xo-reflect -> xo-indentlog2`, an edge
created by converting xo-reflect earlier the same day.

**That path is not a library dependency.** `xo-ratio`'s library declares only
`xo_reflectutil` and `xo_flatstring` (`xo-ratio/CMakeLists.txt:34-35`); the
`xo-reflect` edge comes from `xo-ratio/utest/CMakeLists.txt:19`, and
`pkgs/xo-ratio.nix` files it under `# test-only xo dependencies`.

`subsystem-edges` records an edge for any target — library, example or utest
(`xo_dependency_helper`) — so `xo-deps --why` cannot distinguish a test-only
dependency from a real one, and reported a chain that does not exist at library
level. The conclusion drawn was harmless (indentlog2 was declared explicitly in
xo-unit's utest, correct either way), but the reasoning was not, and it was RC
who noticed. **`xo-deps --why` overstates the graph by exactly this much**, which
matters because it is the tool this repo uses to justify structural claims.
Whether the edge file should distinguish test-only edges is a separate question.

### xo-unit also exposed an unrelated build-system bug

Adding one line to `xo-unit/utest/CMakeLists.txt` broke xo-alloc, xo-object,
xo-reactor and xo-kalmanfilter with `cannot find -lxo_flatstring / -lxo_ratio`.

`xo-unit/CMakeLists.txt` called `xo_export_cmake_config()` *before* its
`xo_headeronly_dependency()` calls. That macro generates
`@XO_FIND_DEPENDENCY_BLOCK@` from the target's `xo_deps` property as it runs —
its own comment at `xo_cxx.cmake:1424-1425` says it must come last — so the block
came out empty and consumers lost the imported targets, turning header-only
libraries into `-l` flags. The installed config had survived only because
xo-unit had not been *reconfigured* since its template switched to the generated
block; the utest edit forced the first regeneration.

Fixed by moving the call after the deps.

**CORRECTED, same day: "three more subsystems have this bug" was wrong.** A
scan reported xo-facet, xo-reflectutil and xo-timeutil as also misordered. All
three were false positives — the detection grep matched **commented-out** lines
(`#xo_export_cmake_config(...)`, `#xo_headeronly_dependency(...)`). Re-running
with both patterns anchored at line start and no leading `#` returns nothing.
Plausible because the grep returned real hits at real line numbers; the cheap
falsification was to read the surrounding lines, which only happened when the
edit was about to be made. xo-unit was the only genuine case.

If the ordering constraint deserves protection rather than vigilance, the fix is
to defer block generation to end-of-configure inside `xo_export_cmake_config`,
so call order stops mattering.
