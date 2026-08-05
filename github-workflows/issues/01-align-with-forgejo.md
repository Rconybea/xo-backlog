# 01 — Realign per-subrepo .github workflows with the forgejo pipeline

Status: open
Type: task

Every xo subrepo published to GitHub carries its own
`.github/workflows/*.yml` that builds it standalone. These have drifted badly
and are now actively wrong for several subsystems.

**Measured 2026-08-05:** 24 subrepos carry `.github/workflows`, holding 28
workflow files across 5 different filename conventions:

| filename | count |
|---|---|
| `main.yml` | 16 |
| `xo-cpp-main.yml` | 4 |
| `cmake-single-platform.yml` | 4 |
| `ubuntu-main.yml` | 3 |
| `nix-main.yml` | 1 |

**25 of those 28 files reference `xo-indentlog`. None references `xo-ppsink`.**

## Why they drift

Each workflow hand-enumerates its dependencies as a sequence of clone-and-build
steps:

```yaml
- name: clone xo-indentlog
  uses: actions/checkout@v3
  with:
    repository: Rconybea/indentlog
    path: repo/xo-indentlog
- name: build xo-indentlog
  ...
```

That list is a second, manually-maintained copy of the dependency graph. Every
`xo_dependency()` change silently invalidates it, and nothing detects the
mismatch until the standalone build runs on GitHub — which nobody watches,
because `.forgejo/workflows/ci-cmake.yaml` is the pipeline that actually gates
work.

The ppsink migration has made this concrete rather than theoretical:

- **xo-ratio is red now.** Its utest and example link `libxo_ppsink`, but
  `ubuntu-main.yml:82-90` and `xo-cpp-main.yml:68-75` clone and build
  `xo-indentlog` and never clone `xo-ppsink`. Known and accepted when the
  migration landed; this ticket is where it gets fixed.
- **xo-reflect, xo-refcnt, xo-flatstring** all still clone indentlog after
  being migrated off it. Stale rather than broken — they over-clone — but
  each is one dependency change away from xo-ratio's situation.

## The fix

`xo-build --clone --with-deps <subsystem>` resolves the dependency set from
`xo-cmake/etc/xo/subsystem-edges`, which is the single maintained copy of the
graph. Replacing the hand-written clone steps with one such invocation removes
the duplicate graph entirely, and the workflows stop being able to drift.

Sketch:

```yaml
- name: clone + build xo-cmake          # supplies xo-build and subsystem-edges
  ...
- name: clone deps + self
  run: xo-build --clone --with-deps ${XO_NAME}
- name: configure + build + test
  run: |
    xo-build --with-deps --configure --with-utests --build --install ${XO_NAME}
    ctest --test-dir ${XO_NAME}/.build --output-on-failure
```

**`xo-build` reads the INSTALLED `subsystem-edges`** (`xo-cmake/bin/xo-build.in:266`),
not the one in the source tree — so xo-cmake must be built and installed before
`--with-deps` sees anything. This has caught us before.

Converging the 5 filename conventions onto one is worth doing in the same pass;
`.forgejo/workflows/ci-cmake.yaml` is the model for structure (matrix over
gcc/clang, `fail-fast: false`, `max-parallel: 1`), though it is not directly
reusable — it builds the whole umbrella from one checkout, whereas these must
assemble a subset from separate GitHub repos.

## Sequencing

Do xo-ratio first: it is the only one currently red, and it is the smallest
end-to-end proof that the `--with-deps` shape works on GitHub's runners. Then
sweep the rest.

Check before relying on it: **xo-ppsink must actually be pushed to
`github.com/Rconybea/xo-ppsink`.** The subrepo remote is configured
(`xo-ppsink/.gitrepo`), but a workflow cloning a repo that was never pushed
fails in a confusing way.

**Files:**
- `xo-*/.github/workflows/*.yml` — 28 files, 24 subrepos
- `.forgejo/workflows/ci-cmake.yaml` — the structural model
- `xo-cmake/bin/xo-build.in:266` — where the installed edges are read
- `xo-cmake/etc/xo/subsystem-edges` — the graph, hand-maintained on purpose

**Done when:**
- no `.github` workflow hand-enumerates a dependency clone list
- xo-ratio's standalone GitHub build is green
- a subsequent `xo_dependency()` change requires no `.github` edit

## Notes

`subsystem-edges` is hand-maintained deliberately — the generated
`.build/xo-subsystem-edges.txt` under-reports when config-time switches are off,
so it must only ever be refreshed from a fully-enabled build. Do not add a
workflow step that regenerates it. See `.scratch/subsystem-edges/issues/`.
