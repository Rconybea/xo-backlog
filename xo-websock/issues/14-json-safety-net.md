# 14 -- a safety net for json output changes: golden snapshot, browser tests in-repo

Status: open
Type: task
Milestone: reflection-driven-json

Every later step of `reflection-driven-json` changes what the server prints.
Each change should show as an explicit diff against a checked-in expectation,
not be inferred from browser tests passing.

- **A golden json snapshot.** Build a server in a deterministic state (fixed
  endpoints, sessions, subscriptions), print it, and compare against a
  checked-in file. Since `.xo-backlog/xo-printjson/issues/02`, ids are
  per-print sequence numbers, so the output is stable. Values that are not
  stable, such as ports and refcounts held by the test, need redacting, as
  `flywheel_frame.test.cpp` does with `ADDR` / `INT`.
- **The introspect browser tests, in the repo.** 19 headless-chrome tests
  and a ref-resolution check (every `_ref_` names an `_id_` in the same
  snapshot) live only in a Claude session scratchpad. They need a home (e.g.
  `xo-websock/example/introspect/test/`) and a runner. They need node and
  headless chrome, so they likely run outside ctest's default set.

**Done when:** both exist in the repo, and a deliberate change to one
printer shows as a reviewable golden diff.
