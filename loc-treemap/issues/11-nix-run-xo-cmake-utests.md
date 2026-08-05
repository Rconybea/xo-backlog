# 11 — Make the nix build actually run the xo-cmake utests

Status: open
Type: task

Ticket 01 wired `xo-cmake/utest` into ctest behind `if (ENABLE_TESTING)`, and it
passes under a plain cmake build. **The nix build never runs it.**
`pkgs/xo-cmake.nix` passes no `cmakeFlags` and sets no `doCheck`, so
`ENABLE_TESTING` is off, `add_subdirectory(utest)` is skipped, and
`nix-build -A xo.cmake` reports success having executed zero of the 102 tests.

This is the same shape as the wider gap: **35 of 62 `pkgs/xo-*.nix` pass
`-DENABLE_TESTING`**, so 27 do not. This ticket fixes only `xo-cmake`, which is
the one this feature added tests to; the sweep across the rest is separate.

**Files:**
- Modify: `pkgs/xo-cmake.nix`

**Interfaces:**
- Consumes: `xo-cmake/utest/CMakeLists.txt` (ticket 01), the `ENABLE_TESTING` guard
  at the end of `xo-cmake/CMakeLists.txt`
- Produces: `nix-build -A xo.cmake` running `utest.xo-loc`

---

- [ ] **Step 1: Confirm the gap before changing anything**

```bash
nix-build -A xo.cmake 2>&1 | grep -i "test" || echo "NO TESTS RAN - gap confirmed"
```

Expected: no test output. If tests *do* run, this ticket is already resolved —
stop here.

- [ ] **Step 2: Follow the house pattern**

`pkgs/xo-cmake.nix` is currently the minimal form — `{ stdenv, cmake }`, no
`lib`, no `doCheck`. The established pattern is `pkgs/xo-arena.nix:24,33-36`:

```nix
  doCheck ? true,
...
    cmakeFlags = [ ... ] ++ lib.optionals doCheck ["-DENABLE_TESTING=1"];
    inherit doCheck;
```

Apply that here, adding `lib` to the argument set. Note the existing packages
spell it `-DENABLE_TESTING=1`, not `=ON`.

- [ ] **Step 3: Give the check phase a python3**

This is what makes `xo-cmake` different from the C++ packages, and the reason
this is a ticket rather than a one-liner: `utest.xo-loc` shells out to
`python3`, which the C++ builders never needed and the nix sandbox does not
supply. Without it `find_program(XO_PYTHON3_EXECUTABLE ...)` fails and — since
ticket 10 changed that branch from `FATAL_ERROR` to a `message(WARNING)` — the
build **succeeds while silently registering no test**, which is exactly the
failure mode this ticket exists to remove.

Add `python3` (the repo already threads `python3Packages` through
e.g. `pkgs/xo-pyutil.nix:5`) to the check phase. `pkgs/*.nix` currently uses
neither `nativeCheckInputs` nor `checkInputs`, so confirm which the pinned
nixpkgs accepts — `nativeCheckInputs` on 23.05+, `checkInputs` on older — and
prefer `nativeCheckInputs` if available.

- [ ] **Step 4: Decide whether scc joins the sandbox**

`TestSccIntegration` is guarded by `@unittest.skipUnless(shutil.which("scc"))`,
so without `scc` those cases **skip silently** — including the
generated-marker guard that ticket 06 added specifically to fail loudly if
`xo-facet/codegen/*.j2` is reflowed.

Two defensible answers; pick one and write the reason into the file:

- add `scc` to the check inputs, so the marker guard runs in CI; or
- leave it out, accepting that the guard is a dev-shell check only, and rely on
  `shells.nix` carrying `scc`.

Note that the walk root in the sandbox is the `xo-cmake` source only, not the
umbrella, so `REPO_ROOT` in `test_xo_loc.py` will not resolve to a tree with
`xo-facet/` in it. Verify what those two tests actually do under nix before
choosing — they may need skipping on a different condition rather than enabling.

- [ ] **Step 5: Verify**

```bash
nix-build -A xo.cmake 2>&1 | tail -30
```

Expected: the check phase runs `utest.xo-loc` and reports
`100% tests passed`. A build that passes without naming the test has not fixed
anything — re-read step 3.

Then confirm the escape hatch still works:

```bash
nix-build -A xo.cmake --arg doCheck false
```

- [ ] **Step 6: Commit**

```bash
git add pkgs/xo-cmake.nix
git commit -m "nix: run xo-cmake utests under doCheck [BUILD]"
```

## Comments

Found by the code review at the end of the xo-loc implementation (tickets
01–10). Deliberately not fixed there: it adds a `python3` dependency to a
package whose closure is currently just cmake, which is a packaging decision
rather than part of the feature.

Related: the wider "many `pkgs/*.nix` run zero tests" sweep — 27 of 62 still
pass no `-DENABLE_TESTING`. This ticket is scoped to `xo-cmake` alone.
