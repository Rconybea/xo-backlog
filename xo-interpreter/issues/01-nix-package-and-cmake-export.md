# 01 — xo-interpreter has no nix package, and its cmake export is incomplete

Status: open
Type: bug

Two related gaps, both surfaced while adding a transitional xo-indentlog
dependency during the xo-expression migration
(`.xo-backlog/xo-expression/issues/01-ppsink-migration-pilot.md`, step 2).

## 1. It is the only buildable subsystem with no `pkgs/*.nix`

Measured 2026-08-08:

```bash
for s in $(xo-build --list | tr ' ' '\n' | grep '^xo-'); do
    [ -f "$s/CMakeLists.txt" ] || continue     # skip empty placeholder subrepos
    [ -f "pkgs/$s.nix" ]       || echo "$s"
done
# => xo-interpreter
```

The `CMakeLists.txt` guard matters: `xo-equable2` and `xo-hashable2` also have
no package, but they are empty placeholder subrepos (README + `.gitrepo`, no
build files) and are not buildable at all — they also abort
`xo-build --all`. Without the guard the command reports three names and the
claim above looks false.

It is also absent from `ci.nix`. Consequences:

- **No nix coverage at all.** `nix-build` is the only check in this tree that
  exercises an *installed* package config the way a real consumer does — it is
  what caught the `-lindentlog` failure and the `xo-tokenizer` example that the
  umbrella build missed. xo-interpreter gets none of that.
- Gap (2) below has therefore never been detectable.

Note `xo-interpreter2` **is** packaged, so the absence looks like an oversight
when xo-interpreter was added rather than a decision.

## 2. Its exported config omits its external dependencies

```
xo_interpreterTargets.cmake:
  INTERFACE_LINK_LIBRARIES "xo_object;xo_expression;xo_reader;indentlog;replxx::replxx;Threads::Threads;subsys"

xo_interpreterConfig.cmake:
  find_dependency(xo_object) / xo_expression / xo_reader / indentlog / subsys
```

`replxx::replxx` and `Threads::Threads` are exported but never resolved. The
mechanism is `xo_external_target_dependency` (`xo_cxx.cmake`):

```cmake
macro(xo_external_target_dependency target pkg pkgtarget)
    find_package(${pkg} CONFIG REQUIRED)
    target_link_libraries(${target} PUBLIC ${pkgtarget})
endmacro()
```

It calls `find_package` and links, but **never appends to the target's
`xo_deps` property** — so `@XO_FIND_DEPENDENCY_BLOCK@` cannot see it. Same for
the raw `target_link_libraries(${SELF_LIB} PUBLIC Threads::Threads)` at
`src/interpreter/CMakeLists.txt:30`.

This is the general problem in
`.xo-backlog/generated-find-dependency/issues/01-external-find-package-deps.md`;
this ticket is one concrete instance with a known fix.

**Latent today, not broken:** nothing depends on xo-interpreter
(`xo-deps --users-of=xo-interpreter --format=names -q` returns nothing), so
there is no consumer to trip it. It fires the moment one appears — and, unlike
the `-lindentlog` case, `replxx::replxx` contains `::`, so CMake raises a *hard
error* at generate time rather than degrading to `-lreplxx`.

### The fix has a working precedent

`xo-kalmanfilter/cmake/xo_kalmanfilterConfig.cmake.in` keeps a manual
`find_dependency` beside the generated block, with a comment saying why:

```cmake
# NOT generated: these arrive via find_package(), not xo_dependency(),
# and xo_export_cmake_config() only knows about xo deps.
find_dependency(Eigen3)

@XO_FIND_DEPENDENCY_BLOCK@
```

`xo-websock` does the same for Libwebsockets. xo-interpreter wants
`find_dependency(replxx)` and `find_dependency(Threads)` in that position.

## For the nix package

`pkgs/xo-interpreter2.nix` is the closest template — same shape, and already
takes `replxx`. `replxx` is in nixpkgs (measured: `replxx-0.0.4`) and is already
used by `pkgs/{xo-interpreter2,xo-tokenizer2,xo-reader,docker-xo-builder}.nix`.

Dependencies to declare, from `src/interpreter/CMakeLists.txt`:

| dep | kind |
|---|---|
| `xo_object`, `xo_expression`, `xo_reader` | xo, PUBLIC |
| `indentlog` | xo, PUBLIC — **transitional**, see below |
| `subsys` | xo, header-only |
| `replxx` | external |
| `Threads` | external (`find_package(Threads REQUIRED)`, line 16) |

Also build `example/replxx/` (`xo_interpreter_replxx`), an executable — worth
having in the package since nix builds examples by default and that is how the
`xo-tokenizer` example regression was caught.

**Sequencing note:** the `indentlog` dep is transitional, added so xo-expression
could drop its PUBLIC propagation. It retires when xo-interpreter migrates to
xo-ppsink. Whoever writes the nix package should expect to change that line
again shortly — not a reason to wait, but worth knowing.

**Files:**
- `pkgs/xo-interpreter.nix` — to create; template `pkgs/xo-interpreter2.nix`
- `ci.nix` — add the attribute
- `xo-interpreter/cmake/xo_interpreterConfig.cmake.in` — add the two
  `find_dependency` lines above the generated block
- `xo-interpreter/src/interpreter/CMakeLists.txt:16,29,30` — where the external
  deps are declared

**Done when:**
- `nix-build ci.nix -A xo-interpreter` succeeds, with examples
- the installed `xo_interpreterConfig.cmake` resolves every name in
  `xo_interpreterTargets.cmake`'s `INTERFACE_LINK_LIBRARIES`

## Notes

Verify by diffing exactly those two lines in the **installed** package, not by
checking that the build passes — an incomplete config builds fine in-tree,
because the umbrella never reads the exported config:

```bash
P=$(nix-build ci.nix -A xo-interpreter --no-out-link)
grep -n find_dependency          $P/lib/cmake/xo_interpreter/xo_interpreterConfig.cmake
grep -n INTERFACE_LINK_LIBRARIES $P/lib/cmake/xo_interpreter/xo_interpreterTargets.cmake
```
