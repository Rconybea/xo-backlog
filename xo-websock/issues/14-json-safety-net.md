# 14 -- a safety net for json output changes: golden snapshot, browser tests in-repo

Status: done 2026-10-05 -- umbrella `619860f3`, `c078aa4f`
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

## Done, 2026-10-05 -- umbrella `619860f3`, `c078aa4f`

**Golden snapshot.** `live-server-snapshot-matches-golden`, in
`utest/WebserverLive.test.cpp` (it needs real connections, so it runs in
utest.websock.live).
- **State:** a stream endpoint with a receiver, an http endpoint, and two
  clients, the second subscribed. The test waits on each step, then prints
  the server. Every websock printer appears.
- **Redaction:** four values vary from run to run and become
  `"<redacted>"`: `listen_port`, and the `_members_` named `listen_port_`,
  `port_`, `mount_origin_` and `output_buf_`.
- **Stability:** 20 runs in a row all matched.
- **Golden file:** `utest/golden/server-snapshot.json`, 716 lines,
  styled one field per line.
  - On a mismatch, the test writes `server-snapshot.actual.json` in its
    working directory and names both files.
  - `XO_UPDATE_GOLDEN=1 utest.websock.live "[golden]"` rewrites the golden
    file. `XO_WEBSOCK_UTEST_SOURCE_DIR` (`utest/CMakeLists.txt`) tells the
    test where the source tree is.

**Browser tests, in the repo** (`example/introspect/test/`):
- the 19 page tests and `refs_resolve.mjs`, unchanged except that
  `refs_resolve` now exits nonzero on failure;
- **`run.sh SERVER [TEST..]`:**
  - it finds chrome (`$CHROME`, then google-chrome, chromium,
    /opt/google/chrome/chrome) and starts it headless on a free CDP port;
  - it starts a fresh server on a free port for each test (`--src-tree`
    for rows and receiver), and absorbs the differences between tests'
    arguments;
  - it exits 1 on any failure; logs and screenshots go to a temp dir.

  20 / 20 pass in about 2.5 min.
- **ctest:** `browser.introspect`, behind the CMake option
  `XO_ENABLE_BROWSER_TESTS` (default OFF: CI has no node or chrome),
  with a timeout of 900 s. It passes in 151 s.

Checked: ctest 49 / 49 with the option OFF, and `xo-build --sweep`.

Not done: the tests repeat about 15 lines of CDP connection code each.
Factoring it into a shared `cdp.mjs` was left out on purpose, so the move
stayed pure.
