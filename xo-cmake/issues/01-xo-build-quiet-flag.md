# 01 — investigate `-q|--quiet` for xo-build

Status: open (investigation)
Type: feature

`xo-build` has **no verbosity control** (`xo-build --help` lists none), and its
output is dominated by noise that is only interesting when something fails.

## Measured 2026-08-08

| | lines |
|---|---|
| full-tree build+install (61 subsystems, with utests + examples) | **3060** |
| of which cmake `-- ` status lines | **2892 (94.5%)** |
| no-op rebuild of a single subsystem (`xo-ppsink`) | 23 |
| lines matching error/warning in a clean build | **0** |

So on success essentially everything printed is noise, and the signal — "did it
work" — is the exit status alone.

Note the noise is **not** xo-build's own: it has only 16 `echo` statements
(`grep -cE '^\s*echo ' xo-cmake/bin/xo-build.in`). It comes from cmake.

## Why this is worth a flag rather than a redirect

The obvious workaround is `xo-build ... >/dev/null 2>&1`, and it is a trap. It
also discards the *error* output, so a failing build looks exactly like a
succeeding one unless the caller separately checks `$?` — and if the caller then
runs tests, `ctest` happily executes the **previous** binary.

That is not hypothetical: it happened during the xo-expression/xo-reader
migration on 2026-08-08 and cost a long debugging detour. Symptoms looked like a
dispatch bug in ppsink — a marker string compiled into the library was not
appearing in rendered output — and the actual cause was a compile error in the
test file, invisible because the build output had been redirected to
`/dev/null`. ctest was running a stale binary against stale assertions.

The desired contract, and the thing a redirect cannot provide:

> **quiet on success, loud on failure** — suppress progress chatter, but always
> surface errors and always fail loudly on a non-zero exit.

## What would have to be suppressed

Three distinct sources, each needing a different lever — this is why it is an
investigation and not a one-liner:

| source | example | candidate lever |
|---|---|---|
| install messages | `-- Installing: /home/roland/local/include/...` | `-DCMAKE_INSTALL_MESSAGE=NEVER` (or `LAZY`) |
| configure status | `-- [process] find_package(reactor) (xo_dependency_helper)` | these are xo's own `message(STATUS ...)` in `xo_cxx.cmake`; would need a gate |
| build progress | `[ 50%] Building CXX object ...` | `cmake --build . -- --no-print-directory`, or make `-s` |

The middle row is the interesting one: those come from xo's own macros, so
quieting them is a decision about xo-cmake's diagnostics, not just a cmake flag.
Worth checking how many of those `message(STATUS ...)` calls are genuinely
diagnostic vs vestigial before adding a gate.

## Questions to settle

- **Does `-q` mean "suppress stdout" or "suppress everything below warnings"?**
  The second is more useful and harder — cmake does not cleanly separate them.
- **Should it still print a one-line per-subsystem summary?** For `--all` over
  61 subsystems, total silence makes progress invisible on a long build. A
  single `xo-build: xo-foo ok` line per subsystem would be ~61 lines instead of
  3060, and is arguably the right default rather than a flag.
- **Should failure re-print what was suppressed?** If `-q` buffers and dumps on
  failure, the caller gets quiet success *and* full diagnostics — strictly
  better than either extreme. Costs buffering.
- **Interaction with `--utest`:** `xo-build.in:556` already re-runs failed tests
  with `--output-on-failure`, i.e. the "quiet until it matters" idea is present
  for tests. Worth matching that pattern.

**Files:**
- `xo-cmake/bin/xo-build.in` — argument parsing, and the 16 `echo` sites;
  `:556` is the existing quiet-until-failure precedent for tests

**Done when:**
- a decision is recorded here, and either implemented or declined with reasoning
- if implemented: a clean full-tree build prints little, a failing one is
  impossible to mistake for success

## Notes

Compare `xo-deps -q` (added 2026-08-08), which suppresses only its summary line
and deliberately leaves errors on stderr, so a mistyped argument cannot be
mistaken for a legitimate negative result. Same principle, much smaller surface.
See `.xo-backlog/CONVENTIONS.md` for why that distinction mattered there.
