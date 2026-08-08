# 02 — Hand-written Config.cmake.in files drift from CMakeLists

Status: open
Type: task

**43 of 60** `xo-*/cmake/*Config.cmake.in` templates are hand-written rather than
using `@XO_FIND_DEPENDENCY_BLOCK@`. Measured 2026-08-08, **17 of them disagree
with the `xo_dependency()` calls in their own `CMakeLists.txt`.**

Two of the 17 were **not latent** — see "How it actually fails" below.

This is the drift the generated block exists to prevent. See
`.xo-backlog/generated-find-dependency/issues/01-external-find-package-deps.md`
for the blocker that stops a mechanical sweep; this ticket records the damage and
two further hazards found while converting four of them.

## Why it is invisible until it isn't

**In-tree builds never read the exported config.** The umbrella build uses
targets directly, so a wrong `Config.cmake.in` costs nothing there. It only
breaks *consumers* — standalone `xo-build`, nix, and anyone doing
`find_package()` against an installed tree.

Concretely (2026-08-08): `xo-alloc/cmake/xo_allocConfig.cmake.in` still said
`find_dependency(indentlog)` weeks after xo-alloc migrated to ppsink. The
umbrella was green throughout; `nix-build ci.nix -A xo-ordinaltree` failed with

```
Could not find a package configuration file provided by "indentlog"
Call Stack:
  .../xo-alloc/lib/cmake/xo_alloc/xo_allocConfig.cmake:30 (find_dependency)
```

Same class as the xo-unit and xo-object cases found earlier in the migration.

## How it actually fails — two distinct symptoms

A missing `find_dependency(X)` produces **one of two** failures, and which one
you get depends on whether `X` is a package the consumer separately resolves:

1. **Configure-time**, `Could not find a package configuration file provided by
   "X"` — when some *other* config in the chain does call `find_dependency(X)`
   and it isn't installed. This is the xo-alloc case above.
2. **Link-time**, `cannot find -lX` — when nothing resolves `X` at all. The
   exported `XTargets.cmake` still lists `X` in `INTERFACE_LINK_LIBRARIES`, so
   CMake sees a name that is not a target, decides it must be a plain library,
   and emits `-lX`. **Header-only deps make this look absurd** — the error
   claims the linker cannot find a library that was never supposed to exist.

Symptom 2 is the one that reads as a mystery. Recorded here so the next
occurrence is diagnosed in one step:

```
ld: cannot find -lindentlog: No such file or directory   # nix-build ci.nix -A xo-process
```

`processTargets.cmake` said `INTERFACE_LINK_LIBRARIES "reactor;printjson;indentlog"`
while `processConfig.cmake` said only `find_dependency(reactor)` +
`find_dependency(printjson)`. Diagnose by diffing exactly those two lines in the
*installed* package:

```bash
P=$(nix-build ci.nix -A xo-process --no-out-link)
grep -n find_dependency          $P/lib/cmake/process/processConfig.cmake
grep -n INTERFACE_LINK_LIBRARIES $P/lib/cmake/process/processTargets.cmake
```

They must agree. The generated block makes them agree by construction.

**Why it survived so long:** `xo_dependency()` is PUBLIC, so `indentlog` reached
xo-process transitively through `reactor` for as long as reactor declared it.
Once reactor migrated to ppsink, xo-process's *own* (correct, explicit)
`xo_dependency(${SELF_LIB} indentlog)` became load-bearing — and its config had
never listed it. The migration didn't introduce the bug; it removed the
transitive path that was hiding it.

## The 17 disagreements

Migration-caused, **fixed 2026-08-08**:

| subsystem | was | now |
|---|---|---|
| xo-alloc | `indentlog`, no ppsink | generated block |
| xo-kalmanfilter | no ppsink | generated block (+ `Eigen3` kept manually) |
| xo-ordinaltree | no ppsink | generated block |
| xo-pyunit | `indentlog`, no ppsink | hand-written, corrected — see hazard 2 |
| xo-process | missing `indentlog` — **hard link failure** | generated block |
| xo-simulator | missing `indentlog` — same | generated block |

Both of the last two also had a `note:` comment naming the wrong CMakeLists
(`simulatorConfig.cmake.in` pointed at `xo-reactor/...`), which is its own
argument for not hand-maintaining these.

Pre-existing drift, **still open** — re-measured 2026-08-08, now **12**, all
py\* wrappers:

```
xo-pydistribution missing xo_distribution, xo_pyutil
xo-pyexpression   missing refcnt          stale xo_pyreflect
xo-pyjit          missing refcnt          stale xo_pyexpression
xo-pykalmanfilter missing xo_kalmanfilter
xo-pyprintjson    missing printjson
xo-pyprocess      missing process
xo-pyreactor      missing indentlog, reactor
xo-pyreflect      missing reflect, xo_pyutil
xo-pysimulator    missing simulator
xo-pywebsock      missing websock, xo_pyutil
xo-pywebutil      missing webutil
xo-callback                               stale refcnt
```

These are **mostly** latent because a python extension module is a leaf — almost
nothing calls `find_package()` on one. The exception is py→py edges:
`xo-pyjit` does `find_dependency(xo_pyexpression)`, which loads a config that
omits `refcnt`. Check that pair first.

`stale` entries (a `find_dependency` for something no longer declared) are
benign while the package is still installed, and become a configure-time
failure the moment it isn't. `xo-ratio` and `xo-unit` also carry stale entries.

**Scan them with SELF_LIB only.** A first pass that also matched `${SELF_EXE}`
reported `xo-interpreter2 missing xo_indentlog2` — a false positive: that
dependency is on the `skrepl` executable, which is never exported.

## Two further conversion hazards (new)

Both produce a **silently empty** dependency block — the template converts
cleanly, the build succeeds, and consumers break later. Ticket 01's external-dep
problem is the third member of this family.

### Hazard 1 — ordering

`xo_export_cmake_config()` snapshots the target's accumulated `xo_deps` **at the
point it is called**. If it appears before the `xo_dependency()` calls, the block
is empty.

`xo-ordinaltree/CMakeLists.txt` had exactly that: export at line 35, deps at
43-47. Converting it produced an empty block; fixed by moving the export call
below the deps, with a comment. The constraint is documented in
`xo_cxx.cmake`'s comment on `XO_FIND_DEPENDENCY_BLOCK`, but nothing enforces it.

**Worth considering:** have `xo_export_cmake_config()` warn (or fail) when the
target exists but `xo_deps` is empty apart from the self-edge. That turns a
silent, deferred failure into a configure-time message.

### Hazard 2 — target name must equal `${PROJECT_NAME}`

The block is read via `get_target_property(${projectname} xo_deps)`. If the
library target has a different name, there is no such target and the block is
empty.

`xo-pyunit`: `project(xo_pyunit)` but `set(SELF_LIB pyunit)`. Left hand-written
with a comment saying why. Any subsystem whose `SELF_LIB` differs from its
`PROJECT_NAME` is unconvertible until one is renamed or the macro takes the
target name explicitly.

Worth auditing which other subsystems have that mismatch before the next sweep.

## Also noted

`pkgs/*.nix` had the same drift a layer up — 16 files still naming
`xo-indentlog`, 19 missing deps the subsystem genuinely declares. Synced
2026-08-08 from `subsystem-edges`. Nix's `propagatedBuildInputs` are transitive,
so the packaging was *inaccurate but working*: xo-ratio built fine while naming
`xo-indentlog`, because ppsink reached it via xo-flatstring. Left alone
deliberately: the "extra" deps a naive diff flags, since removing them risks
breaking transitive paths for no benefit.

**Files:**
- `xo-*/cmake/*Config.cmake.in` — 41 hand-written, 12 still disagreeing
- `xo-cmake/cmake/xo_macros/xo_cxx.cmake` — `xo_export_cmake_config`, where a
  warn-on-empty check would go
- `pkgs/*.nix` — synced, but will drift again by the same mechanism

**Done when:**
- every `Config.cmake.in` either uses the generated block or documents why it
  cannot
- the remaining 12 disagreements are resolved
- an empty generated block is detected rather than shipped

## Notes

Verify conversions by **reading the generated `Config.cmake`**, not by checking
that the build passes — a wrong or empty block builds fine in-tree. The cheap
end-to-end check is `nix-build ci.nix -A <consumer>`, which exercises the
installed config the way a real consumer does.
