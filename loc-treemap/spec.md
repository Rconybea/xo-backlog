# LOC treemap visualization (`xo-loc`)

Status: spec — design agreed, ready for implementation planning
Type: feature spec

## Summary

A treemap of the XO codebase: one tile per `xo-*` subsystem, **area ∝ lines of
code**, **color = artifact kind**. Sibling to `xo-deps` — same "visualize the
subsystem world" family, same packaging (`configure_file`'d script installed to
bindir), same CLI ergonomics.

Unlike `xo-deps`, this tool has **no checked-in snapshot data**: it runs the
counter live on each invocation. That sidesteps the subsystem-edges staleness /
xo-cmake-hash-churn problem entirely (see `.scratch/subsystem-edges/issues/01,02`).
A rarely-changing `.in` script on xo-cmake's hash path is acceptable; data churn
is not.

## Decisions

### Backend: `scc`, not `cloc` — settled by measurement

Evaluated `scc` 3.5.0 against `cloc` 2.04 on this tree:

| | scc 3.5.0 | cloc 2.04 |
|---|---|---|
| whole tree (64 dirs, ~2.7k files) | **37 ms** | ~0.15 s for *one* subsystem |
| complexity | per-file int + `WeightedComplexity` | none |
| COCOMO | `-f json2` → cost / months / people | none |
| generated-file detection | `--gen` + `--generated-markers` | none |
| per-file records | `--by-file -f json` includes `Location` | `--by-file` includes path |
| gitignore-aware | yes | **no** |

The decisive properties are the last two. `Location` carries the full path, so a
**single** `scc` invocation over the repo root can be aggregated by first path
component — no per-subsystem loop. And gitignore-awareness matters a lot here:
cloc counted 165 files in `xo-unit`, of which 66 were `.build/` artifacts.

`--format=json2` additionally exposes `estimatedCost` / `estimatedScheduleMonths`
/ `estimatedPeople`. Not used by the default view; noted as available.

### The walk must start at the repo root

`scc` only honours `.gitignore` files at or below its walk root. Invoking
`scc xo-alloc` — the obvious way to implement `xo-loc xo-alloc` — misses the
umbrella `.gitignore` and re-admits ~28k LOC of build junk, which also
materialises phantom languages:

- CMake depfiles (`*.o.d`) detected as **D** — 13,247 LOC
- `CMakeFiles/CMakeConfigureLog.yaml` as **YAML**
- `CMakeCache.txt` as **Plain Text**
- `CMakeFiles/*/CompilerIdC/CMakeCCompilerId.c` as **C**

Walking from the repo root yields **zero** `.build` files and none of the phantom
languages. Only `xo-unit` escaped this in testing, because it happens to carry its
own `.gitignore`; the other subsystems rely on the umbrella one.

**Rule: always walk from the repo root. Subsystem arguments filter the output,
never the walk.**

### Vendored third-party: hard-excluded

`xo-imgui/include/imgui/` is vendored Dear ImGui — **63,732 LOC, 31% of the whole
tree**. `imgui.cpp` alone is 11,375 lines, larger than any XO subsystem. Left in,
the map is mostly a picture of somebody else's library.

Vendored paths are dropped at aggregation. `xo-imgui` then reads as 11,011 LOC
(its genuine `include/xo/` + `example/` content). `--include-vendored` restores
them.

The vendored path list is a small default baked into the script
(`xo-imgui/include/imgui/`), overridable via `--vendored=PATH,...`.

### Generated code: excluded by default, `--include-generated` restores

Generated files carry a marker on line 3:

```
 *  Generated automagically from ingredients:
```

so `scc --gen --generated-markers "Generated automagically"` detects them
natively: **369 files, 10,968 LOC (7.2% of the tree)**.

A filename regex on the `IFoo_DBar.{hpp,cpp}` convention was considered and
**rejected — it is wrong in both directions**:

- 8 files matching the pattern are **hand-written** (`@author Roland Conybeare,
  Dec 2025`), e.g. `xo-alloc2/include/xo/alloc2/arena/IAllocator_DArena.hpp`,
  `xo-gc/src/gc/IAllocator_DX1Collector.cpp` — the regex would falsely brand them
  generated.
- 106 genuinely generated files **do not** match it, e.g.
  `xo-type/include/xo/type/Type.hpp` and all of `xo-printable2/detail/*`.

Marker detection gets both right and needs no regex maintenance.

Generated share is very uneven, which is why it must not be silently folded into
Code:

(LOC here is the subsystem total *including* generated files, since that is what
the percentage is taken against; the default view's tiles are correspondingly
smaller.)

| subsystem | LOC incl. generated | generated |
|---|---|---|
| xo-printable2 | 258 | 50% |
| xo-procedure2 | 3,000 | 36% |
| xo-type | 2,094 | 34% |
| xo-expression2 | 6,228 | 29% |
| xo-object2 | 3,412 | 25% |
| xo-reader2 | 15,835 | 22% (3,458) |
| xo-facet | 2,264 | 22% |

**Data-model consequence:** the original sketch derived kind from a
language→kind map. `Generated` cannot come from language — a generated `.hpp` is
still C++. It is a **per-file boolean orthogonal to language**, so the normalized
table needs a `generated` column, not just a language lookup.

### Artifact kind

Kind is the color dimension. Language → kind:

| kind | languages |
|---|---|
| Code | C++, C++ Header, C Header, C, Objective C++, Python, JavaScript, GLSL, LLVM IR |
| Build | CMake, Nix, Makefile, Autoconf, BASH, Shell, YAML, INI, Patch |
| Docs | ReStructuredText, Markdown, Plain Text, License, HTML |
| Data | JSON5, JSON, Jinja, SVG |
| Generated | *(per-file flag, overrides the language mapping)* |

Unmapped languages fall back to an explicit **`Other`** kind and the tool emits a
warning naming them — so a newly-introduced language shows up as unclassified
rather than being silently miscategorised.

Kind already earns its place in the default view: `xo-cmake` is 100% Build
(3,767 LOC) and every `xo-py*` binding is dominantly Build — e.g.
`xo-pykalmanfilter` is 206 Code against 539 Build. A LOC-only map would assert
those are real code.

### Which subsystems

`xo-*` glob, **directories only** — the root walk otherwise picks up `pkgs`,
`docs`, `.forgejo`, `README.md`, `shells.nix`, and `xo-deps.svg`, which is a
*file* matching `xo-*`.

Positional arguments restrict the drawn set, `cloc`-style: `xo-loc xo-gc xo-arena`.

### Empty placeholders: reported, not drawn

`xo-hashable2` and `xo-equable2` contain only `.gitrepo` + a one-line `README.md`
— 1 LOC each. They are placeholder subrepos not yet filled in.

Subsystems under `--min-loc` (default 20) are **listed in the footer, not drawn**.
Drawing them — especially at a min-area floor — would assert content that does
not exist.

The test is on **total** LOC, not code LOC. Testing "zero code LOC" wrongly sweeps
up `xo-cmake`, a real 3,767-LOC subsystem that is 100% Build.

### Layout: nested two-level, with a minimum-area floor

Two levels: subsystem → kind sub-tiles.

Area is 2D, so the long tail is less dire than a LOC ranking suggests. With
vendored and generated excluded and the two placeholders dropped, the default view
is **133,784 LOC across 62 drawn subsystems**, top tile `xo-reader2` at 9.3% (down
from imgui's 31%). On a 1200×800 canvas the smallest drawn subsystem,
`xo-symboltable` (109 LOC), is already **782 px²** — roughly 30×26, visible
without any intervention.

A minimum-area floor is therefore polish, not rescue, and it is cheap:

| floor F | subsystems floored | total area inflation |
|---|---|---|
| none | 0 | — |
| **173.45** (derived, see below) | **3** | **0.09%** |
| 250 | 10 | 0.5% |
| 1000 | 30 | 14.5% |

Scheme: `weight_i = max(loc_i, F)`, area ∝ weight. The floored three are
`xo-symboltable` (109), `xo-printable2` (130) and `xo-pywebsock` (159).

**F is derived, not hardcoded.** The floor is expressed as "minimum tile = one
label box (≈64×18 px)" and solved as

```
F = T·a / (1 − N·a)
```

where `T` = total LOC of drawn subsystems, `N` = number of drawn subsystems, and
`a` = minimum tile area as a fraction of canvas area (`64·18 / (width·height)`).
So the floor self-adjusts to canvas size and subsystem count rather than being a
magic constant.

**Nesting rule:** a floored tile is by construction exactly big enough for its own
label, so subdividing it would immediately re-hide it. A tile is therefore
subdivided into kind sub-tiles only when

```
tile_area ≥ k · A_min
```

where `k` is the number of distinct kinds present in that subsystem and `A_min` is
the minimum legible tile area (`64·18` px). Otherwise it renders **flat, colored
by dominant kind**. Large subsystems nest; the tail renders flat — which is where
each is readable anyway.

### Complexity: a color mode, plus always-on detail

Area always means LOC — the chart's premise stays fixed. Color is the swappable
dimension:

- `--color=kind` (default) — categorical over artifact kind
- `--color=complexity` — sequential scale over cx/kLOC, **applied to Code tiles
  only**; non-Code tiles grey out, since scc complexity is ~0 for Docs/Data/Build
  languages

cx/kLOC carries real signal independent of size — in the default view it spans
10 → 161, separating branch-heavy code (`xo-unit` 161, `xo-arena` 153,
`xo-ordinaltree` 149) from declarative/template-heavy code (`xo-interpreter2` 42,
`xo-expression2` 50).

Regardless of color mode, the label/tooltip always carries LOC, complexity,
cx/kLOC, file count, and the language breakdown.

### Renderer: Python 3 stdlib → SVG + HTML

A single `python3` script, `configure_file`'d and installed to bindir exactly like
`xo-deps.in`.

Two facts settled this:

1. **scc's CSV cannot carry the design.** Its columns are
   `Language,Provider,Filename,Lines,Code,Comments,Blanks,Complexity,Bytes,ULOC`
   — there is no `Generated` column. `--no-gen` can make scc *drop* generated
   files, but CSV can never say *which* files were generated, so the
   `--include-generated`-as-its-own-color view is impossible from CSV. JSON is
   required, which rules out an awk pipeline.
2. **Python 3 is already a first-class dependency of this repo.**
   `xo-facet/codegen/genfacet` is `#!/usr/bin/env python3`, the umbrella
   `CMakeLists.txt:155` has an `xo_umbrella_genfacet_all` target, and
   `shells.nix:55-60` already ships `python3Packages.{python,jinja2,json5,pybind11}`.
   The "adds a python dep" objection applies only to matplotlib/squarify/plotly —
   python3 **stdlib** is free.

Squarify needs recursive float math and text metrics; stdlib Python does that in
~100 lines, and recursion makes the two-level nesting natural.

Formats:

- `svg` (default) — self-contained, hand-emitted
- `html` — that same SVG plus inlined vanilla JS for hover / click-to-zoom,
  mirroring `xo-deps --format=html`
- `csv` / `json` — the normalized table, echoing `xo-deps --format=dot` (emit the
  intermediate)

**No `png`.** `xo-deps` gets it free from graphviz; a treemap tool would need
rsvg or inkscape, and that dependency is not worth it.

Palette values (kind categorical colors, complexity sequential scale, light/dark
behavior) are pinned during implementation via the `dataviz` skill rather than
guessed here. The SVG must set an **explicit background fill** — `xo-deps.in:196`
already records that inheriting the viewer's background renders elements
near-invisible.

## Architecture

Five stages, each with one job:

| stage | input → output |
|---|---|
| **collect** | `scc --by-file --gen --generated-markers "Generated automagically" -f json <root>` → raw JSON. One invocation. |
| **normalize** | raw → rows `(subsystem, language, kind, generated, vendored, loc, complexity, nfiles)` |
| **select** | apply exclusions + subsystem args → grouped `(subsystem, kind)` weights |
| **layout** | `[(key, weight)] × rect → [(key, rect)]`, squarified. Knows nothing about SVG. |
| **render** | svg / html / csv / json emitters over the laid-out rects |

`normalize` is the **pluggable-backend seam**: a `cloc` adapter emits the same
table with `complexity=0` and `generated=False`, and nothing downstream knows
which counter ran.

`layout` is a **pure function** — which is what makes it testable, and makes
nesting simply a second call on each subsystem's rect.

### Root discovery

The walk must start at the repo root, but the tool is installed to bindir. It
mirrors `xo-deps`'s discovery ladder:

1. `--root=DIR`
2. `$XO_SOURCE_ROOT`
3. `git rev-parse --show-toplevel`
4. upward search from `$PWD`

## CLI

```
xo-loc [options] [SUBSYSTEM...]
  --format=svg|html|csv|json    Output format (default: svg)
  --output=FILE                 Output path (default: xo-loc.<format>)
  --color=kind|complexity       Color dimension (default: kind)
  --include-generated           Draw generated code as its own color
  --include-vendored            Draw vendored third-party code
  --vendored=PATH,...           Override the vendored path list
  --kinds=KIND,...              Restrict to these kinds; any of
                                Code, Build, Docs, Data, Generated, Other
  --min-loc=N                   Below this, report but do not draw (default: 20)
  --width=N --height=N          Canvas size; the min-tile floor derives from these
  --root=DIR                    Source root (default: auto-discover)
  --scc=PATH                    scc binary (default: scc; used by tests)
  --list                        List subsystems and exit
  -h, --help
```

Listing `Generated` in `--kinds` implies `--include-generated`; likewise the
vendored path list is only consulted when `--include-vendored` is given.

## Error handling and honesty

- Missing `scc` → the `command -v` check `xo-deps.in:125` already uses, with an
  install hint.
- **The tool always prints what it hid.** A treemap that silently drops a third of
  the tree is this design's main failure mode, so every exclusion is reported:

  ```
  excluded 369 generated files (10,968 LOC)
  excluded vendored: xo-imgui/include/imgui/ (63,732 LOC)
  2 subsystems empty, not drawn: xo-hashable2, xo-equable2
  min-area floor applied to 3 subsystems (+0.09% area)
  ```

- Unmapped languages are named in a warning rather than silently bucketed.

## Testing

- **`layout`** — real unit tests against a pure function: exact tiling of the
  parent rect, bounded aspect ratios, area conservation under the min-area floor.
- **`normalize`** — tests against a small checked-in `scc` JSON fixture. The
  fixture doubles as the pinned backend contract, so an scc upgrade that changes
  the JSON shape fails loudly.
- **`render`** — smoke test: well-formed SVG, expected tile count.

## Packaging

`xo-cmake/bin/xo-loc.in`, `configure_file`'d and installed to bindir like
`xo-deps.in`. No checked-in data file.

## Reference: sibling tool

`xo-cmake/bin/xo-deps.in` — 376 lines of bash; discovers its edge list, emits
svg/png/dot/html via graphviz, `--focus/--deps-of/--users-of`, self-contained HTML
with embedded JS. Follow its CLI style and ergonomics; the renderer does not carry
over, since treemaps are not graphviz.

## Measurements referenced above

Captured on this tree with scc 3.5.0, walking from the repo root:

- raw total, root walk: **204,822 LOC**, 0 `.build` artifacts, ~37 ms
- vendored Dear ImGui: **63,732 LOC** (31%)
- generated: **369 files, 10,968 LOC** (7.2%)
- default view: **133,784 LOC across 62 drawn subsystems** (64 `xo-*` dirs − 2 placeholders)
- largest tile: `xo-reader2` 12,377 LOC (9.3%)
- smallest drawn: `xo-symboltable` 109 LOC (782 px² on 1200×800)

## Comments
