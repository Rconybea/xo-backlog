# Decouple subsystem orchestration data from xo-cmake's build-input hash

Status: open (deferred)
Type: task
Priority: primary fix (demotes 01)

## Problem (root cause)

Editing `xo-cmake/etc/xo/{subsystem-list,subsystem-edges}` triggers a
full-from-scratch rebuild of the whole umbrella in CI (>60min duty cycle), even
though the edit changed no code. Consequence: snapshot updates get intentionally
batched/deferred to let a CI cycle finish — which is the origin of the staleness
tolerance and the re-snapshot churn tracked in [01].

### Mechanism (confirmed)

- `pkgs/xo-cmake.nix` is a single-output derivation with `src = ../xo-cmake`.
  Nix hashes `src`, so ANY file change under `xo-cmake/` — including pure data
  files — yields a new xo-cmake store path, and every derivation that build-depends
  on xo-cmake rebuilds. (A multi-*output* split does NOT help: `src` still rehashes,
  so `out` gets a new path even with identical contents. Only physically moving the
  data off `src`, or content-addressed derivations, breaks the cascade.)
- Nothing build-consumes the *content* of these files. Their only readers are
  xo-cmake's human/orchestration tools (`xo-build`, `xo-deps`, `xo-gen-clang-format`)
  and the umbrella's top-level emit — none run during a subsystem compile. The data
  is pure tooling data sitting on the universal-input hash path.

## Fix direction

Relocate `subsystem-list` + `subsystem-edges` into a standalone leaf derivation
that nothing build-depends on. Re-snapshotting then rehashes only that leaf →
zero downstream rebuild → eager (even automatic) re-snapshot becomes viable →
the staleness risk in [01] mostly dissolves and its write-side guard becomes
optional insurance rather than a necessity.

## Obstacles to design around

- **Docker bootstrap:** `xo-docker-build` bind-mounts all of `xo-cmake/` source
  into the builder and re-bootstraps an identical xo-cmake inside the container.
  Moving the data out means that bootstrap must now also carry the separate data
  package.
- **Install block:** `xo-cmake/CMakeLists.txt` installs `etc/xo/subsystem-{list,edges}`
  to `$DATADIR/etc/xo`; the tools bake `@CMAKE_INSTALL_FULL_DATADIR@/etc/xo/...`.
  The relocated package must install to the same runtime path so the baked paths
  still resolve.
- **cmake (Forgejo) CI** may invalidate differently than nix; confirm the cascade
  source there too before assuming the nix fix covers both.

## Relationship

- Primary fix. Demotes [01] (snapshot-publish-guard): once re-snapshot is free,
  eager re-snapshot removes most of 01's risk.

## Comments
