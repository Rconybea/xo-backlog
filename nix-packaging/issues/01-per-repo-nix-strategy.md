# 01 — Decide how individual xo repos should provide nix packaging

Status: open
Type: decision

Not a defect to fix, a policy to settle. Right now there are two nix packaging
trees, they have forked, and only one of them is maintained.

Measured 2026-08-06.

## What exists

**`xo-unit` is the only subrepo that packages itself.** It is the sole holder of
a `flake.nix` and a `pkgs/` directory among all the subrepos:

```
xo-unit/flake.nix
xo-unit/pkgs/   -- 10 .nix files
pkgs/           -- 64 .nix files (umbrella)
```

Both descend from the same original pattern. **All ten of xo-unit's files have
diverged from their umbrella counterparts**, by 28 to 122 differing lines --
nothing is still in sync.

## The fork is structural, not drift

The two variants do genuinely different things, so they cannot simply be deduped.
Taking `xo-subsys.nix`, the smallest divergence, as representative:

| | `xo-unit/pkgs/` | umbrella `pkgs/` |
|---|---|---|
| source | `src = fetchGit { url = "https://github.com/rconybea/subsys"; }` | `src = ../xo-subsys` |
| dependencies | `propagatedBuildInputs = [ xo-indentlog ]` | `[ xo-ppsink ]` |
| cmake flags | `-DXO_CMAKE_CONFIG_EXECUTABLE=...` | `-DCMAKE_MODULE_PATH=...` |
| xo-cmake | not in `nativeBuildInputs` | in `nativeBuildInputs` |

So: the umbrella variant was adapted to **local sibling paths** and has been kept
current; xo-unit's **fetch-from-GitHub** variant froze. Source provenance is a
real functional difference -- a standalone repo has no siblings to point at.

## Why it matters now

Both trees hand-maintain a copy of the inter-subsystem dependency graph, so both
drift silently as `xo_dependency()` calls change. Currently:

- **20 of the umbrella's 64 package files still name `indentlog`; only 7 name
  `ppsink`** -- and this is the *maintained* tree.
- `xo-unit/pkgs/xo-flatstring.nix` still lists `propagatedBuildInputs = [ xo-indentlog ]`,
  long after xo-flatstring was migrated off it.

This is the same failure mode as the per-subrepo `.github` workflows (see
`.xo-backlog/github-workflows/issues/01-align-with-forgejo.md`): a second,
manually-maintained copy of the dependency graph that nothing in-tree validates.
The nix case is worse only in that there are two copies rather than one.

## Options

- **(a) Umbrella only.** Nix packaging is an umbrella concern; individual repos
  build with plain cmake, which is what `.forgejo/workflows/ci-cmake.yaml`
  already exercises. Delete `xo-unit/flake.nix` and `xo-unit/pkgs/`.
- **(b) Generate per-repo packaging** from one source of truth, parameterised on
  source provenance (local path vs `fetchGit`). The dependency lists come from
  `xo-cmake/etc/xo/subsystem-edges` rather than being retyped.
- **(c) A shared nix expression set** consumed by both umbrella and standalone
  repos, with provenance injected by the caller.
- **(d) Status quo, extended** -- hand-write `pkgs/` for more subrepos. Listed
  for completeness; it multiplies the drift.

## Questions to settle first

1. **Is standalone nix packaging wanted at all**, or is nix an umbrella-level
   convenience? If plain cmake is the supported standalone path, (a) is nearly
   free and removes a stale tree.
2. If wanted, **for which repos** -- all of them, or the leaf libraries people
   actually consume standalone (xo-unit, xo-ratio, xo-flatstring)?
3. **Does anything currently consume `xo-unit/flake.nix`?** If it has no users,
   that settles it. Worth checking before doing any work.

**Files:**
- `xo-unit/flake.nix`, `xo-unit/pkgs/*.nix` — the standalone tree (10 files)
- `pkgs/*.nix` — the umbrella tree (64 files)
- `xo-cmake/etc/xo/subsystem-edges` — the maintained dependency graph both
  duplicate

**Done when:**
- a policy is chosen and recorded
- there is exactly one hand-maintained copy of the dependency graph, or zero

## Notes

Independent of the choice, the umbrella's `pkgs/` is stale with respect to the
indentlog→ppsink migration (20 files still name indentlog). Whether that gets
fixed by hand or falls out of (b)/(c) depends on the decision, so it is not worth
patching piecemeal first.
