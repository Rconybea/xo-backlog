# 06 — xo_external_target_dependency() should never link PUBLIC

Status: open
Type: refactor / packaging
Raised: 2026-09-26, from `.xo-backlog/xo-websock/issues/02`

`xo_external_target_dependency(target pkg pkgtarget)`
(`xo-cmake/cmake/xo_macros/xo_cxx.cmake:1702`) is exactly:

```cmake
find_package(${pkg} CONFIG REQUIRED)
target_link_libraries(${target} PUBLIC ${pkgtarget})
```

PUBLIC re-exports the external package's ENTIRE link interface to every
consumer of the xo library. That is how xo-websock/02 went wrong: libwebsockets'
exported target lists openssl (by full path) in its public link interface, so
every library linking xo-websock recorded direct `NEEDED libssl.so.3` entries it
never used. Once installed, the loader resolved them through a runpath that
lacked openssl's directory, and `import xo.websock` failed from `~/local`.
xo-websock was fixed by hand: libwebsockets linked PRIVATE, its include
directories re-exported PUBLIC. This ticket makes that the macro's behaviour.

## Survey: no current call site needs its package linked publicly

58 calls as of 2026-09-26. The 50 on executables (Catch2 41, replxx 5, CLI11 4)
are unaffected: nothing links against an executable. The 5 on libraries:

| call site | pkg headers in this subsystem's PUBLIC headers? | consumers need |
|---|---|---|
| xo-kalmanfilter / `Eigen3::Eigen` | yes -- `KalmanFilterInput.hpp`, `KalmanFilterTransition.hpp`, `print_eigen.hpp` | include dirs; nothing to link (Eigen is `INTERFACE IMPORTED`) |
| xo-testutil / `Catch2::Catch2` | yes -- `UtestListener.hpp`, `try_test_array.hpp` | include dirs; nothing to link (`INTERFACE IMPORTED`) |
| xo-websock / `jsoncpp_lib` | no -- `Webserver.cpp` only | nothing |
| xo-interpreter / `replxx::replxx` | no -- its `.cpp` only | nothing |
| xo-pykalmanfilter / `Eigen3::Eigen` | no -- a pybind module | nothing |

```bash
grep -rn "xo_external_target_dependency(" --include=CMakeLists.txt . | grep -v '\.build/' \
  | grep -vE "SELF_EXE|UTEST_EXE|SELF_EXECUTABLE_NAME" | grep -v ':\s*#'
# per subsystem: does the package leak into public headers?
grep -rlE '#include\s*<(Eigen/|catch2/|json/|replxx)' xo-*/include
```

(The xo-websock row describes the jsoncpp call as it stands. The libwebsockets
call was already replaced by hand in xo-websock/02.)

So the axis that actually varies is **whether the package's HEADERS reach
consumers**, not whether its linkage does.

## Proposal

1. **Always link PRIVATE.**
2. **An opt-in to re-export the package's compile-time usage requirements**
   (include dirs, and anything else a consumer needs to compile against it)
   WITHOUT its link interface -- for the subsystems whose public headers include
   the package (kalmanfilter, testutil, and libwebsockets in xo-websock).
3. **When (2) is used, record the package** so the generated
   `@XO_FIND_DEPENDENCY_BLOCK@` emits `find_dependency(${pkg} CONFIG)`: an
   exported usage requirement that names the imported target needs consumers
   to re-find the package. Today that is hand-listed per `*Config.cmake.in`,
   and was forgotten for jsoncpp (xo-websock/02). Same move as
   `project_generated-find-dependency`, extended to external packages -- today
   the `xo_deps` property only carries xo targets.

Mechanism for (2), UNVERIFIED -- worth a spike first:

- **`$<COMPILE_ONLY:${pkgtarget}>`** (CMake >= 3.27; this tree uses 3.31):
  `target_link_libraries(${target} PRIVATE ${pkgtarget} INTERFACE $<COMPILE_ONLY:${pkgtarget}>)`
  propagates every usage requirement except linking. It is the purpose-built
  tool, and covers compile definitions/options as well as include dirs. Check
  that it survives `install(EXPORT)` intact.
- **Fallback:** what xo-websock does by hand today --
  `target_include_directories(${target} PUBLIC $<TARGET_PROPERTY:${pkgtarget},INTERFACE_INCLUDE_DIRECTORIES>)`.
  Carries include dirs only. Sufficient for libwebsockets (its target sets only
  includes + link libs), not proven for the others.

```bash
d=$(grep -h '^Eigen3_DIR' .build/CMakeCache.txt | cut -d= -f2)
grep -n 'INTERFACE_' $d/*Targets*.cmake        # what Eigen's target actually carries
```

Open decisions:

- keyword name for (2) -- e.g. `xo_external_target_dependency(tgt pkg pkgtgt PUBLIC_HEADERS)`
- change the existing macro's semantics in place, or add a new macro and retire
  the old one. In place is one edit per call site at most (only the two that
  need headers), but it silently changes behaviour everywhere at once.

## Risks to check before switching

- **A consumer may rely on the old leak.** Something that includes
  `<json/json.h>` or `<replxx.h>` directly while declaring only xo-websock or
  xo-interpreter would stop compiling or linking. Umbrella builds cannot catch
  this -- one cmake context, the package already found -- only `xo-build
  --sweep` and `nix-build` can. As of 2026-09-26 both greps below come back
  EMPTY (no downstream user includes either header), so the source-level risk
  is nil today; the link-level one is what the sweep is for:
  ```bash
  for s in $(xo-deps --users-of=xo-websock --format=names -q); do
      grep -rlE '#include\s*<json/' $s --include=*.hpp --include=*.cpp; done
  for s in $(xo-deps --users-of=xo-interpreter --format=names -q); do
      grep -rlE '#include\s*<replxx' $s --include=*.hpp --include=*.cpp; done
  ```
- **Static and header-only xo targets.** Every current library call site is a
  shared library (`xo_add_shared_library4`) or a pybind module. For a STATIC
  library, PRIVATE still exports `$<LINK_ONLY:...>`, which needs
  `find_dependency` too. An INTERFACE xo target cannot take PRIVATE at all. The
  macro should either handle both or refuse them loudly.
- **xo-websock's hand-written pieces go away:** the libwebsockets block in
  `xo-websock/src/websock/CMakeLists.txt`, and both hand-listed
  `find_dependency` lines in `xo-websock/cmake/websockConfig.cmake.in`.
  jsoncpp would become fully private and need no `find_dependency` at all.
  The `--as-needed` block stays: it keeps libwebsock.so's OWN NEEDED list
  clean, which PRIVATE linkage does not.

## Done when

- the macro never links PUBLIC, and has the opt-in for exporting compile-only
  requirements
- the five library call sites are converted; kalmanfilter and testutil (and
  libwebsockets in xo-websock) use the opt-in
- external packages exported via the opt-in appear in the generated
  `@XO_FIND_DEPENDENCY_BLOCK@`; xo-websock's hand-listed lines are removed
- no installed xo library carries a spurious NEEDED entry from an external's
  link interface:
  ```bash
  for f in ~/local/lib/lib*.so; do
      n=$(readelf -d $f | grep -cE 'NEEDED.*(libssl|libcrypto|libjsoncpp|libreplxx)')
      [ "$n" -gt 0 ] && echo "$f $n"; done
  ```
  (libwebsock.so legitimately needs libjsoncpp, so expect exactly that line)
- `xo-build --sweep` ok in both stages, and `nix-build` ok for kalmanfilter,
  testutil, websock and interpreter plus one consumer of each
