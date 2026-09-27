# 07 — split xo-cmake: the bottom layer vs the parts that know the subsystem topology

Status: open
Type: refactor / build
Raised: 2026-09-27 (RC)

xo-cmake is a dependency of every xo subsystem, AND it knows the list of all
xo subsystems and the edges between them. Topologically it is both the lowest
subsystem and the highest. Split it: the part that genuinely belongs at the
bottom of the hierarchy, and the parts that depend on
`etc/xo/{subsystem-list, subsystem-edges}`.

## Measured, 2026-09-27

What reads the topology files (`grep -rln "subsystem-list\|subsystem-edges" xo-cmake`):

| file | uses |
|---|---|
| `xo-cmake/etc/xo/subsystem-list` (74 lines) | the subsystems, in build order |
| `xo-cmake/etc/xo/subsystem-edges` (226 edges) | dependency edges, tsort format |
| `xo-cmake/etc/xo/dot-clang-format` | include-order priorities -- derived from the edges |
| `bin/xo-build.in` | installed `subsystem-list` / `subsystem-edges`: sweep order, `--deps` closures |
| `bin/xo-deps.in` | installed `subsystem-edges` (fallback: `.build/subsystem-edges`) |
| `bin/xo-gen-ci.in` | generates `.forgejo/workflows/ci-cmake.yaml` from `subsystem-list` |
| `bin/xo-gen-clang-format.in` | generates `.clang-format` from `subsystem-edges` |
| `bin/xo-cmake-config.in` | `--subsystem-list`: reports the installed path |
| `share/xo-macros/xo-reconfigure.in` | `--capture-subsystem-edges` publishes a build's edges INTO xo-cmake |
| `CMakeLists.txt` | installs the three `etc/xo/` files |
| `cmake/xo_macros/xo_cxx.cmake` (~line 1090) | WRITES `subsystem-edges` from each build's dependency declarations (a mechanism, not a consumer) |

The umbrella's own `CMakeLists.txt` (around line 197) shares its build order
with `xo-cmake/etc/xo/subsystem-list` and `ci.nix`.

The rest -- `cmake/xo_macros/` (`xo_cxx.cmake`, `xo-project-macros.cmake`,
`code-coverage.cmake`), `xo-cmake-config`, the coverage harnesses, `xo-loc`,
`xo-python`, `scaffold-subdir`, the `share/xo-macros` templates -- is what
every subsystem needs to BUILD, and knows no other subsystem.

## Why it matters (measured)

- **Every topology edit rebuilds everything under nix.** `pkgs/xo-cmake.nix`
  builds from `src = ../xo-cmake`, and 77 `pkgs/*.nix` depend on xo-cmake
  (`grep -ln "xo-cmake" pkgs/*.nix | wc -l`). Adding a subsystem edits
  `etc/xo/subsystem-list` -- 8 commits in its recent history
  (`git log --oneline -- xo-cmake/etc/xo/subsystem-list`) -- which changes
  xo-cmake's store hash, and every package that builds with it rebuilds.
- **Reinstall churn in the sweep path**: xo-cmake must be reinstalled when the
  subsystem list changes (a known step for `xo-build --sweep`), because the
  installed tools read the installed copies.
- **The cycle is conceptual, not only a cost**: the bottom layer cannot be
  understood, versioned, or cached independently of the whole tree above it.

## Proposal (sketch)

- **xo-cmake (bottom):** the cmake macros, `xo-cmake-config`, templates,
  coverage harnesses -- no knowledge of which subsystems exist.
  `xo_cxx.cmake`'s edge CAPTURE stays here: it records a build's declared
  dependencies without knowing the others.
- **A new top-level package** (name open -- e.g. `xo-topology`, `xo-tree`):
  `subsystem-list`, `subsystem-edges`, `dot-clang-format`, and the tools that
  operate on the whole tree -- `xo-build` (sweep), `xo-deps`, `xo-gen-ci`,
  `xo-gen-clang-format`, and `xo-reconfigure --capture-subsystem-edges`'s
  publishing step. Nothing builds against it; developer tools and CI use it.

## Decided (RC, 2026-09-27)

- **The topology files live in the new package** (`subsystem-list`,
  `subsystem-edges`, `dot-clang-format`), not at the umbrella root.
- **xo-build is split.** Single-subsystem behaviour (configure / build /
  utest / install of ONE subsystem) stays in xo-cmake; the whole-tree tool
  (sweep order, dependency closures) lives in the new package and INVOKES the
  single-subsystem tool for each subsystem.
- **xo-websock issue 12's pieces:** the per-target type-map generator is
  bottom (xo-cmake); the merge over a subsystem set's dependency closure is
  top (the new package).

## Open (remaining)

- **Name** of the new package.
- **Ordering of the move**: nix packaging (`pkgs/xo-cmake.nix` + a new pkg),
  the docker CI image, `~/local` installs, and every `xo-cmake-config
  --subsystem-list` caller change together.

## Done when

- xo-cmake contains nothing that names or enumerates other subsystems
- the topology files and whole-tree tools live in their own package, which no
  subsystem build depends on
- editing `subsystem-list` / `subsystem-edges` no longer changes xo-cmake's
  nix store hash (so no longer rebuilds every package)
- `xo-build --sweep`, `xo-deps`, `xo-gen-ci`, `xo-gen-clang-format` work as
  before; both CI pipelines green
