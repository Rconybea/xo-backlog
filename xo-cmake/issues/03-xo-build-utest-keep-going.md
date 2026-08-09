# 03 — `xo-build --utest` fails fast, so a sweep cannot use it

Status: diagnosed
Type: feature

`xo-build --utest` accepts several subsystems and reports one line each, but it
**stops at the first failing one**. That makes it unusable for the whole-tree
verification sweep, which is the one place a test runner most needs to survey
rather than abort.

Measured 2026-08-08:

```bash
xo-build -q --utest xo-ppsink xo-object
#   xo-build: xo-ppsink ok (utest)
#   xo-build: xo-object ok (utest)
#   rc=0

xo-build -q --utest xo-jit xo-ppsink
#   xo-build: unit tests failed -- re-running failures with --rerun-failed --output-on-failure
#   <full jit trace>
#   rc=1
#   -- and xo-ppsink NEVER RAN: no status line, no 'utest.ppsink' anywhere in
#      the output
```

This tree has a **permanent** known failure — `xo-jit machpipeline.fptr`,
`.xo-backlog/xo-jit/issues/01` — so `xo-build -q --utest $SUBS` over the 61
subsystems would stop at xo-jit and leave every subsystem after it untested,
while still looking like it ran the tests. That is the failure mode `-q` was
designed to prevent (`.xo-backlog/xo-cmake/issues/01`), reappearing one level
up: not "a failure looks like a success", but "a partial run looks like a full
one".

## Proposal

`-k | --keep-going`, for `--utest` at least: run every named subsystem, report
per-subsystem status, exit non-zero if any failed.

Fail-fast is right for `--configure/--build/--install` — subsystems are built in
dependency order, so failures after the first are mostly noise. It is wrong for
`--utest`, where the point is a survey and every subsystem is independent.

Worth deciding whether `-k` should be the default for `--utest` rather than a
flag. The argument for: nobody running tests over a list wants to be told about
only the first failure. The argument against: consistency with the other
phases, and a single-subsystem invocation is unaffected either way.

**Output volume needs a thought.** On failure `--utest` re-runs with
`--rerun-failed --output-on-failure` and that output goes straight to the
terminal (by design, ticket 01). For one subsystem that is exactly right; for a
61-subsystem sweep with `-k`, several failures would each dump a full trace and
bury the summary. Options: print the per-subsystem summary last, or buffer the
re-runs and dump them after the summary, or gate the detail behind a flag.

## Why this is filed as a tooling gap, not a doc fix

`xo-build`'s job is to capture build patterns so they do not get hand-recreated
at each call site. The evidence that this one is missing:

`.xo-backlog/CONVENTIONS.md` spells the sweep as a shell loop —

```bash
for s in $SUBS; do ctest --test-dir $s/.build; done
```

— which exists only because `--utest` cannot survey. That loop then teaches the
wrong lesson by adjacency: it sits directly under an `xo-build -q ... $SUBS`
line that correctly passes a *list*, and the asymmetry is unexplained.

Observed cost, 2026-08-08, by an agent that had used `-q` correctly all session
and had read both CONVENTIONS.md and ticket 01:

1. wanted a per-subsystem status summary for two subsystems
2. did not recall that `xo-build` already prints one line per subsystem — the
   feature ticket 01 added for exactly this
3. so wrote `for s in ...; do xo-build ... done`, pattern-matched from the ctest
   loop above
4. each iteration then printed its own status line, duplicating the loop's echo
5. so redirected to `/dev/null` to suppress the duplicate
6. thereby discarding diagnostics — **the precise anti-pattern ticket 01 exists
   to prevent**, arrived at from the opposite direction

Step 6 is what the documented rule prohibits; step 3 is where the leverage is.
A rule that says "never redirect" does not survive a caller who has an
unsatisfied need for a compact multi-subsystem summary — they will rebuild it
by hand and re-derive the redirect. Give `--utest` the survey behaviour and the
loop, the redirect, and the rule all become unnecessary together.

**Files:**
- `xo-cmake/bin/xo-build.in` — `--utest` handling, and the existing
  `--rerun-failed --output-on-failure` re-run
- `.xo-backlog/CONVENTIONS.md` — the sweep recipe, which should collapse to two
  `xo-build` lines with no shell loop once this lands

**Done when:**
- `xo-build --utest a b c` runs all three even when `a` fails, and exits
  non-zero
- the summary survives being read after several failures
- CONVENTIONS.md's verification recipe contains no hand-written `ctest` loop

## Notes

Related: ticket 01 (`-q`) established *quiet on success, loud on failure* by
buffering rather than filtering. The same instinct applies here — a survey that
buffers per-subsystem detail and prints it under a summary is the natural
extension, not a new mechanism.
