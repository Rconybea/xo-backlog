# 03 — `xo-build --utest` fails fast, so a sweep cannot use it

Status: fixed 2026-08-09
Type: feature

## Resolution (2026-08-09)

`-k | --keep-going` implemented in `xo-cmake/bin/xo-build.in`, as designed
below. Verified on this tree:

```bash
xo-build -q -k --utest xo-jit xo-ppsink
#   xo-jit FAILED, xo-ppsink still ran ("ok (utest)"), rc=1, 0 skipped
#   -- before, xo-ppsink never ran at all

# with xo-alloc deliberately broken:
xo-build -q -k --build xo-alloc xo-object xo-ppsink
#   FAILED   xo-alloc (build)
#   skipped  xo-object (depends on xo-alloc)     <- downstream, pruned
#   xo-ppsink ok (build)                         <- independent, still built
#   rc=1, and the actual compiler error still printed in full

xo-build -q --build xo-alloc xo-object xo-ppsink   # no -k: unchanged
#   stops at xo-alloc, no summary, rc=2
```

The whole-tree sweep is now one command with no shell loop, and
`.xo-backlog/CONVENTIONS.md` has been updated to it.

### Also fixed: zero tests no longer counts as a pass

Surfaced by the first green sweep, which claimed **60 passed** where the old
loop said 34. `ctest` exits 0 on "No tests were found!!!", so 26 of the 61
subsystems were being counted as passes having run nothing — this ticket's own
complaint ("a partial run looks like a full one") reappearing in the number the
ticket exists to make trustworthy.

Now reported as a third outcome:

```
xo-build: xo-ppsink    ok (utest)
xo-build: xo-allocutil ok (utest:no-tests)
xo-build: 1 failed, 0 skipped, 26 with no tests
```

34 + 26 + 1 = 61, and the 34 reconciles with the old loop-based count.

Detected with `ctest -N` ("Total Tests: N"), which runs nothing. That is
ctest's own structured summary rather than an error-message shape, so it is not
the output-filtering ticket 01 argues against. **An unparseable count falls
through and runs the tests** — doing the work is the safe direction; claiming
"no tests" wrongly is not.

Same defect as `.xo-backlog/nix-packaging` (packages built without
`-DENABLE_TESTING` ran zero tests and looked green). Worth treating a
suspiciously round pass count as a symptom.

### Two bugs found while testing, both on the SUCCESS path

Recorded because neither would have appeared in the failure-path tests this
ticket's design implies, and a `-k` implementation is naturally exercised on
failure:

- **A stray `continue` in the no-tests branch** would have skipped that
  subsystem's remaining phases (`--install`) and its summary line. Having no
  tests is not a failure. Now an `if/else`. The neighbouring `continue` on test
  *failure* is deliberate and stays — do not install a subsystem whose tests
  just failed.
- **`[[ cond ]] && exit 1` as the script's last statement** returns 1 when the
  condition is FALSE, so a clean `-k` run exited non-zero:
  `xo-build -q -k --utest xo-allocutil` reported `0 failed, 0 skipped` and then
  failed. That would have broken CI on a green tree. Now an explicit `if`, plus
  a terminal `exit 0`.

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

## Design (settled with RC, 2026-08-08)

`-k | --keep-going`, an opt-in flag, matching `make -k`: continue with work
that does not depend on what failed.

**Build phases (`--configure` / `--build` / `--install`) prune by dependency.**
When subsystem `foo` fails, drop every remaining subsystem downstream of it —
they cannot build against a broken dependency, and their failures would be
cascade noise. One call per failure:

```bash
xo-deps --users-of=foo --format=names -q      # foo, plus everything downstream
```

**`--utest` continues unconditionally, with no pruning.** A test failure does
not invalidate a dependent's tests: the dependent built fine, and its tests
exercise different code. Pruning there would discard real signal — and would
defeat the case that motivated this ticket, since `xo-jit`'s tests fail
permanently and everything downstream of it should still be surveyed.

One sentence for the asymmetry, if it needs defending later: **a build failure
invalidates dependents; a test failure does not.**

### Details that are easy to get wrong

- **`-q`, not `2>/dev/null`, on the `xo-deps` call.** Exit 2 is "no such
  subsystem"; with stderr hidden it is indistinguishable from exit 1, "nothing
  depends on foo". The pruning would then silently *under*-skip and run
  dependents against a broken dependency. Treat any non-zero `xo-deps` exit as
  "do not prune" — fail toward attempting too much, since that surfaces as
  ordinary build errors rather than as a quiet gap. Same trap CONVENTIONS.md
  records for the `--why` loop.
- **`--users-of`, not `--why`.** `--why=X:Y` exists and answers exactly the
  path question, but it answers it for ONE pair; pruning needs the whole
  downstream set, so `--users-of` is one call per failure where `--why` would
  be one per remaining candidate per failure.
- **A skipped subsystem must not read as passed.** The summary has to
  distinguish `ok` / `FAILED` / `skipped (depends on foo)`. Reporting only
  failures would recreate this ticket's own complaint one level down: a partial
  run looking like a full one.
- **Ordering.** `xonames` is rebuilt bottom-up only on the `--with-deps` /
  `--all` paths; an explicit list is taken as given. The skip set must
  therefore be consulted at the top of the loop rather than assumed to be
  forward-only, so a list in arbitrary order still behaves.

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
- `xo-build -k --utest a b c` runs all three even when `a` fails, and exits
  non-zero
- `xo-build -k --build a b c`, with `b` downstream of a failing `a`, skips `b`
  and still attempts an independent `c`
- skipped subsystems are reported as skipped, naming what they were skipped for
- the summary survives being read after several failures
- CONVENTIONS.md's verification recipe contains no hand-written `ctest` loop

### Where the change goes

The loop and the per-subsystem status line already exist — ticket 01 added
them. Fail-fast comes from exactly two hard exits:

- `run_phase()` in `xo-cmake/bin/xo-build.in` — covers configure/build/install
- the inline `exit 1` in the `--utest` block

Under `-k`, `run_phase` returns the code instead of exiting; its call sites sit
directly in the per-subsystem loop body, so they can `continue` to skip that
subsystem's remaining phases. Failures accumulate; the summary and the non-zero
exit come after the loop.

## Notes

Related: ticket 01 (`-q`) established *quiet on success, loud on failure* by
buffering rather than filtering. The same instinct applies here — a survey that
buffers per-subsystem detail and prints it under a summary is the natural
extension, not a new mechanism.
