# 01 — `xo_export_cmake_config` omits external (non-xo) dependencies

Status: open
Type: task

`xo_export_cmake_config()` generates the installed config's `find_dependency()`
block from the `xo_deps` target property
(`xo-cmake/cmake/xo_macros/xo_cxx.cmake`, the `XO_FIND_DEPENDENCY_BLOCK` loop).
`xo_deps` is only ever appended to by `xo_dependency_helper()`, i.e. by
`xo_dependency()` / `xo_headeronly_dependency()`.

**Anything discovered another way is invisible to it** — `find_package()` called
directly in a subsystem's `CMakeLists.txt`, and `xo_external_pkgconfig_dependency()`.
So converting a hand-rolled `Config.cmake.in` to the generated form **silently
drops that subsystem's external dependencies**.

This is not hypothetical: it was caught during the 2026-08-04 conversion sweep.
`xo-websock` listed `find_dependency(Libwebsockets)` by hand; a naive conversion
would have deleted it. It is currently worked around by keeping the line above
`@XO_FIND_DEPENDENCY_BLOCK@` with a comment marking it exempt — see
`xo-websock/cmake/websockConfig.cmake.in`.

**Why it matters:** it blocks mechanical conversion of the ~50 remaining
hand-rolled templates, which is the whole point of the generated block. And a
silent drop reproduces exactly the failure class the generated block exists to
prevent — an undefined imported target degrading to a raw `-lfoo` at link time.

**Files:**
- Modify: `xo-cmake/cmake/xo_macros/xo_cxx.cmake` (`xo_export_cmake_config`,
  and possibly a new `xo_external_dependency()` macro)
- Reference: `xo-websock/cmake/websockConfig.cmake.in` (the current workaround)

---

- [ ] **Step 1: Survey what is actually external**

Current external packages named in `Config.cmake.in` templates:

| package | subsystems |
|---|---|
| `Eigen3` | xo-flatstring, xo-ratio, xo-kalmanfilter |
| `Libwebsockets` | xo-websock |

Discovery mechanisms in use outside `xo_dependency()`:

- `find_package(LLVM ...)` — xo-jit
- `find_package(Vulkan ...)`, `find_package(Threads ...)` — xo-imgui and others
- `xo_external_target_dependency()` — calls `find_package` and links, but never
  appends to the target's `xo_deps`, so the generated block cannot see it
- `xo_external_pkgconfig_dependency()` — 3 call sites (SDL2, in xo-imgui)

**A concrete instance with a known fix:** `xo-interpreter` exports
`replxx::replxx` and `Threads::Threads` in `INTERFACE_LINK_LIBRARIES` while its
config resolves neither — see
`.xo-backlog/xo-interpreter/issues/01-nix-package-and-cmake-export.md`. Worth
reading alongside this ticket: it is latent only because nothing depends on
xo-interpreter yet, and it shows the failure is *harder* than the `-lfoo` case,
since a name containing `::` makes CMake raise a hard error at generate time
rather than degrading to a link flag.

Note the mechanisms are not equivalent, and the fix should not assume they are.
A `find_package` CONFIG dependency (Eigen3, Libwebsockets) exports an imported
*target* and genuinely needs `find_dependency()` in the installed config. A
pkg-config dependency links a *raw library name* (`SDL2` → `-lSDL2`) and bakes
absolute include paths into the exported target — that needs no
`find_dependency()`, but is brittle in its own way (see
[[project_sdl2-stale-cmake-cache]]). Confirm which category each site is in
before deciding what to emit.

- [ ] **Step 2: Choose the approach**

Three candidates; (c) is worth doing regardless of which of (a)/(b) wins.

- **(a) Record externals in a parallel property.** Add
  `xo_external_dependency(target pkg)` that does the `find_package()` and
  appends to an `xo_external_deps` property; emit it into the same block.
  Makes conversion fully mechanical, but touches every external call site.
- **(b) Keep externals hand-listed above the generated block.** Status quo for
  `xo-websock`. Zero machinery, but leaves a hand-maintained region in a file
  headed "DO NOT EDIT" — the exact ambiguity that caused this ticket.
- **(c) Add a completeness guard.** At `xo_export_cmake_config()` time, diff the
  target's `INTERFACE_LINK_LIBRARIES` against what the generated block covers,
  and `message(WARNING)` (or FATAL) on anything linked but not findable. This
  would have caught the original `xo-expression`/`indentlog` bug **at configure
  time in the umbrella build**, rather than at link time in CI.

- [ ] **Step 3: Implement and convert one real consumer**

Whatever is chosen, prove it on `xo-websock` (`Libwebsockets`) and one Eigen3
consumer, and delete the hand-listed workaround if (a) is taken.

- [ ] **Step 4: Verify against a STANDALONE build, not the umbrella**

This bug class is invisible in the umbrella build, because every xo dependency
is a real in-tree target there. The test that matters is installing to a prefix
and configuring a downstream subsystem against the installed configs:

```bash
cmake --install <build> --prefix /tmp/xo-pfx
cmake -S xo-websock -B /tmp/ws-build -DCMAKE_PREFIX_PATH=/tmp/xo-pfx
cmake --build /tmp/ws-build
```

A green umbrella build proves nothing here.

- [ ] **Step 5: Then resume the conversion sweep**

With externals handled, the remaining hand-rolled templates can be converted.
An audit comparing declared `xo_dependency()` calls against each template's
`find_dependency()` list (excluding commented lines, and checking transitive
reachability rather than mere absence) is the way to find the ones that are
actually broken versus merely redundant.

## Comments

Raised 2026-08-04 while fixing the `-lindentlog` CI failure in xo-jit. Root
cause there was the same two-sources-of-truth split this generated block is
meant to eliminate: `xo-expression` linked `indentlog` (so `install(EXPORT)`
exported it) while its hand-maintained config never declared it, so standalone
consumers got a bare `-lindentlog`. Seven templates were converted then;
`xo-websock` was the one that could not be converted cleanly, which is what
surfaced this gap.

Related: [[project_generated-find-dependency]].
