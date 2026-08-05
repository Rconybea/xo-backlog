# xo-loc Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `xo-loc`, a treemap of the XO subsystem world with area ∝ lines of code and color = artifact kind, installed alongside `xo-deps`.

**Architecture:** A five-stage pipeline in one Python 3 script — `collect` (one `scc` invocation from the repo root) → `normalize` (per-file rows, the pluggable-backend seam) → `select` (exclusions + aggregation) → `layout` (pure squarify, two-level) → `render` (svg/html/csv/json). Layout is a pure function so it carries real unit tests; every exclusion is reported in the output so the map never lies silently.

**Tech Stack:** Python 3 **stdlib only** (`json`, `argparse`, `subprocess`, `csv`, `collections`, `unittest`). External runtime dependency: `scc` ≥ 3.5.0. Build glue: CMake `configure_file`, mirroring `xo-deps.in`.

Spec: [`.xo-backlog/loc-treemap/spec.md`](spec.md)

## Global Constraints

- **Python 3 stdlib only.** No `matplotlib`, `squarify`, `plotly`, `jinja2`, `pytest`, or any other third-party import. `pytest` is *not* installed in this environment; tests use `unittest`.
- **`xo-cmake/bin/xo-loc.in` must be valid Python as it sits in the source tree.** Any `@VAR@` substitution appears only inside a string literal (`XO_LOC_VERSION = "@PROJECT_VERSION@"`), so tests can import the `.in` file directly without a configure step.
- **The `scc` walk always starts at the repo root.** Positional subsystem arguments filter output, never the walk. Walking a subdirectory misses the umbrella `.gitignore` and re-admits ~28k LOC of `.build` artifacts.
- **scc invocation is exactly:** `scc --by-file --gen --generated-markers "Generated automagically" -f json <root>`
- **Generated-file marker string:** `Generated automagically`
- **Default vendored path list:** `xo-imgui/include/imgui/`
- **Kind names, exactly:** `Code`, `Build`, `Docs`, `Data`, `Generated`, `Other`
- **Minimum legible tile:** 64×18 px, so `A_MIN = 1152` px².
- **Default canvas:** 1200×800.
- **Default `--min-loc`:** 20, tested against **total** LOC, never code LOC (code-LOC would wrongly drop `xo-cmake`, a real 3,767-LOC subsystem that is 100% Build).
- **No `png` output format.**
- Run tests from the repo root: `python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v`

## Ticket Index

| # | Ticket | Deliverable |
|---|---|---|
| 01 | [scaffold](issues/01-scaffold-and-test-harness.md) | `xo-loc.in` skeleton + unittest harness + ctest wiring; `--version` works |
| 02 | [normalize](issues/02-normalize.md) | scc JSON → `Row` table; kind mapping |
| 03 | [select](issues/03-select.md) | exclusions + aggregation + honesty report |
| 04 | [squarify](issues/04-squarify.md) | pure squarified layout |
| 05 | [min-area + nesting](issues/05-min-area-and-nesting.md) | derived floor + two-level subdivide rule |
| 06 | [collect](issues/06-collect.md) | root discovery + scc invocation |
| 07 | [CLI + csv/json](issues/07-cli-and-table-formats.md) | **end-to-end working tool** |
| 08 | [SVG renderer](issues/08-svg-renderer.md) | `--format=svg` |
| 09 | [HTML renderer](issues/09-html-renderer.md) | `--format=html`, hover + click-zoom |
| 10 | [install + docs](issues/10-install-and-docs.md) | `configure_file` + install + README |
| 11 | [nix runs the utests](issues/11-nix-run-xo-cmake-utests.md) | follow-up: `pkgs/xo-cmake.nix` executes zero tests |

Tickets 02–05 are pure functions with no I/O and can be implemented in any order.
Ticket 07 is the first point at which the tool runs end to end.

Ticket 11 was raised by the code review after 01–10 were done; it is a packaging
follow-up, not part of the original plan.

## Deferred / follow-up

Two coupling risks were identified during design and deliberately left as
follow-ups rather than blocking this plan:

- The vendored path list is hardcoded knowledge about the tree, living on
  xo-cmake's hash path. If a second vendored library appears, consider
  discovery by convention instead.
- The generated-marker string is coupled to `xo-facet/codegen/*.j2` wording.
  If those templates are reflowed, `xo-loc` silently stops excluding generated
  code. Ticket 06 includes a guard test that fails loudly if the marker matches
  nothing in the real tree.
