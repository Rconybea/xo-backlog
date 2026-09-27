# 09 — a live end-to-end websocket test

Status: open
Type: test
Raised: 2026-09-26, from `.xo-backlog/xo-websock/issues/07` (closed without it)

Everything in `utest.websock` runs below the socket: `WsSessionRouter`,
`UrlRouter`, `WsSessionTable`, `WsSessionSender`, sinks, and a `Webserver` that
is made but never started (`xo-websock/utest/Webserver.test.cpp`). No test
starts a server, connects a client, and reads what arrives. So the parts of
`WebserverImpl` that only run with libwebsockets are untested:

- session open/close bookkeeping (`notify_ws_session_open` / `_close`,
  issue 08's id assignment at `LWS_CALLBACK_HTTP_BIND_PROTOCOL`)
- the outbound path: `send_text` -> `WebsocketSessionRecd` queue ->
  `lws_cancel_service` -> `LWS_CALLBACK_EVENT_WAIT_CANCELLED` ->
  `lws_write_pending_traffic`
- issue 07's unregister drain: `unregister_stream_endpoint` ->
  `removed_endpoint_v_` -> `lws_end_removed_subscriptions` -> each session's
  `end_subscriptions_on`
- issue 05's sender close on session close, observed from outside

## Proposed cases

1. subscribe over a real socket: `subscribed` with a sub_id, then frames in
   `{"stream", "sub_id", "seq", "event"}` envelopes
2. `send` reaches the endpoint's `StreamReceiver`, whose reply arrives on the
   same connection
3. **unregister while subscribed** (issue 07): the client receives
   `{"cmd": "unsubscribed", "sub_id": N, "reason": "endpoint removed"}`, the
   endpoint's unsubscribe runs once, and no frame for N follows the reply
   (frames already queued before it may precede it)
4. **a sink retained past its session** (issues 05, 08): after the client
   disconnects, notifying the kept sink sends nothing -- and a second client,
   connected meanwhile, receives nothing from it
5. two clients get distinct session ids; reconnecting never reuses one

## Needs deciding

- **The client.** Measured 2026-09-26: no python `websockets` /
  `websocket-client` / `aiohttp`, no `websocat` or `wscat` on PATH; node
  v22.19.0 has a global `WebSocket` (`node -e "console.log(typeof WebSocket)"`
  -> `function`). Options: libwebsockets' own client mode inside a C++ test
  (no new dependency, most code); node's `WebSocket` driven from a script
  (node is in the dev shell, not necessarily in CI -- check both
  `docker-xo-builder` and the nix CI); a python client added to the
  environment.
- **The port.** `WebserverConfig::port` goes straight to
  `lws_context_creation_info::port` (`xo-websock/src/websock/Webserver.cpp`).
  Whether port 0 (OS-assigned) works, and how a test would learn the port, is
  NOT verified.
- **Where it lives.** In `utest.websock` (then every ctest run opens a socket
  and starts a thread), or a separate executable / ctest label.
- **A stream endpoint.** C++ can build one directly from a `StreamEndpointDescr`
  whose subscribe function keeps the sink; python needs a reactor source
  (`xo-reactor2websock`). The C++ route is simpler.

Overlaps `.xo-backlog/xo-websock/issues/03` (browser-stepped flywheel demo):
the same started-server-plus-client setup.

## Decided (RC, 2026-09-26)

- client: libwebsockets' own client mode, in C++ (`WsTestClient`)
- port: 0, with a new `Webserver::listen_port()`. Verified from the header:
  `lws-context-vhost.h` documents port 0 as "the kernel will pick a random
  port", read back with `lws_get_vhost_listen_port()`. Client mode is compiled
  in (`LWS_WITH_CLIENT` in the installed 4.3.5 `lws_config.h`).
- a separate executable, `utest.websock.live`, registered with ctest
- the stream endpoint built in C++
- two steps: A = infrastructure + case 1; B = cases 2-5

## Progress

**Step A, 2026-09-26.** Umbrella `fd7c4287` (includes the `stop_webserver`
deadlock fix below).

- `Webserver::listen_port()` (`xo-websock/include/xo/websock/Webserver.hpp`):
  0 until listening and after stopping. Set on the service thread right after
  `lws_create_context`, from `lws_get_vhost_listen_port`.
- `xo-websock/utest/WsTestClient.{hpp,cpp}`: lws client on its own thread;
  `wait_connected` / `send` / `wait_received(n)` / `received` / `close`, every
  wait bounded by a timeout. `send`/`close` wake it with `lws_cancel_service`;
  `EVENT_WAIT_CANCELLED` asks for a writeable callback on the connection.
- `xo-websock/utest/WebserverLive.test.cpp`, case 1
  `live-subscribe-then-frames`: port 0, connect, subscribe, `subscribed` with
  a sub_id, then two events pushed from the TEST's thread arrive as frames with
  the right stream / sub_id / seq / event. The sink is taken from the
  subscribe function via a condition variable, not from the `subscribed`
  reply -- the reply is sent BEFORE the subscribe function runs (issue 06).
- `xo-websock/utest/CMakeLists.txt`: `utest.websock.live` links
  `websockets_shared` itself (websock links it PRIVATE).

**Bug found and fixed: stopping a running server deadlocked.**
`WebserverImpl::stop_webserver` held `mutex_` while calling
`interrupt_stop_webserver`, which locks `mutex_` again (not recursive): the
caller blocked on itself, and the service thread -- out of its loop, since
`interrupt_flag_` was already set -- blocked on the same mutex setting
`state_ = stopped`. Found with gdb (`thread apply all bt`): thread 1 in
`interrupt_stop_webserver` <- `stop_webserver` <- the test's teardown; the
service thread in `run()` taking `mutex_`. Never exercised before -- nothing
started a server in a test, and the demos stop by process exit. Fix: decide
under the lock, interrupt after releasing it. The first run of the live test,
before the fix, is its falsification (hung; killed by `timeout`).

**Race on `lws_cx_`, fixed after `fd7c4287`** (RC: "add it to step A";
landed separately since A was already committed). Umbrella `fd2eecec`.
`interrupt_stop_webserver` and `unregister_stream_endpoint` read `lws_cx_` from
the caller's thread while the service thread wrote it at start and stop: a data
race on the pointer, and a use-after-free window (a non-null read, then
`lws_context_destroy`, then `lws_cancel_service` on freed memory) that an
atomic alone would not close.

- new `std::mutex cx_mutex_`; new `WebserverImpl::wake_service_thread()`, the
  only way other threads reach the context: null check AND
  `lws_cancel_service` under the lock. Safe to hold across the call -- it
  writes a pipe, never blocks or calls back.
- `run()` works on a local context; publishes it to `lws_cx_` under the lock
  after creation, and nulls it under the lock BEFORE `lws_context_destroy`.
- `interrupt_stop_webserver` sets `interrupt_flag_` then wakes; a stop before
  the context is published is still seen, since `run()` checks the flag
  before its first `lws_service`.
- Verified: live test 10/10, utest.websock 39/498, umbrella 49/49,
  `xo-build --sweep` ok. NOT shown race-free: that needs ThreadSanitizer, a
  separate build configuration, not set up.

**Join hang, fixed with it** (RC). If `lws_create_context` failed, `run()`
returned without setting `state_ = stopped`, so `join_webserver()` -- and
`~WebserverImpl`, which joins -- waited forever. Now the failure path sets
`stopped` and notifies, under `mutex_`.

- New live case `live-a-server-that-cannot-start-still-joins`: a second server
  on the first's port. Observed: lws fails to bind (`ERROR on binding ... (-1
  98)`, EADDRINUSE; `Failed to create default vhost`; `lws init failed`). The
  join runs on a DETACHED thread signalling a promise, so a regression fails
  the test instead of hanging it (a `std::async` future would block in its
  destructor). Falsified by removing the fix: fails at the join's
  `wait_for`.
- utest.websock.live 2 cases / 23 assertions, 5/5 runs; utest.websock
  39/498; umbrella 49/49; `xo-build --sweep` ok.

Both fixes (race, join hang) landed together: umbrella `fd2eecec`.

Results: live test passed 5/5 consecutive runs (~0.12 s). Falsified with a
compiling change (`EVENT_WAIT_CANCELLED` not calling
`lws_write_pending_traffic`): fails at its first `wait_received`. Umbrella
ctest 49/49 (`utest.websock.live` new). `xo-build --sweep` ok in both stages;
its xo-websock build registers both executables (`ctest --test-dir
xo-websock/.build -N`). CI not yet observed -- whether localhost sockets work
in both pipelines is unverified until a push.

**Step B, 2026-09-26 -- cases 2-4.** Umbrella `a134c88d`. `xo-websock/utest/WebserverLive.test.cpp` only.

- `live-send-reaches-the-receiver-and-its-reply-comes-back` (case 2): a
  `StreamReceiver` gets the msg as sent; its reply through the handed sink
  arrives as a frame of that subscription.
- `live-unregister-ends-a-live-subscription` (case 3): two subscriptions on
  `/fw/${id}`; `unregister_stream_endpoint` from the test's thread; both get
  `unsubscribed` with `"reason": "endpoint removed"`, in sub_id order; the
  endpoint's unsubscribe ran exactly twice; a later subscribe is
  `unknown stream`. Case 3 as first written also asked that "no frame for N
  follows the reply" -- that is the SOURCE's job once its unsubscribe runs,
  and here the test is the source; asserting the unsubscribe ran is the
  server's half.
- `live-a-sink-kept-past-its-session-reaches-no-one` (case 4): client 1
  subscribes and disconnects; once the server has handled the close (the
  endpoint's unsubscribe ran), client 2 connects; the application pushes to
  the kept sink; client 2's first message is its own `subscribed`, and no
  message carries the stale event.
- Case 5 (distinct session ids) NOT added as a live case: the session id is
  internal -- nothing a client receives carries it -- so observing it would
  need a test-only accessor. Covered by `WsSessionTable.test.cpp` (never
  reused) and, behaviourally, by case 4.

Falsified with compiling changes:
- the service thread not draining removed endpoints: case 3 fails at its
  `wait_received(4)`.
- sender NOT closed on session close, alone: case 4 still PASSES -- ids are
  never reused (issue 08), so the stale send finds no session. Defence in
  depth, as intended.
- sender not closed AND `WsSessionTable::next_id` recycling ids: case 4 fails
  -- the stale frame reaches client 2 ahead of its own reply. Issue 05's
  misdelivery hazard, reproduced over a real socket.

utest.websock.live 5 cases / 65 assertions, 10/10 consecutive runs;
utest.websock 39/498; umbrella 49/49; `xo-build --sweep` ok. CI still not
observed.

**CI, 2026-09-27** (as reported by RC, plus the Forgejo task list): run 506 at
umbrella `b8b8a647` -- which includes `utest.websock.live` (landed
`fd7c4287`, all 5 cases since `a134c88d`) -- `cmake-build (clang)` success;
`cmake-build (gcc)` still running, past the xo-pywebsock stage. The nix
smoke-test for that commit not yet run. Close when both cmake builds and the
smoke-test are green.

**CI, checked 2026-09-27.** Run 506/507 at umbrella `b8b8a647` (after the
live-test commits `fd7c4287`, `fd2eecec`, `a134c88d`): `cmake-build (clang)`
success (task 576), `cmake-build (gcc)` success (577), `smoke-test` (nix)
success (578).

- **cmake pipeline: runs the live test.** Its xo-websock step
  (`.forgejo/workflows/ci-cmake.yaml`, "build xo-websock") is
  `xo-build --configure --with-utests --with-examples` / `--build` /
  `--utest` -- the same path as the local sweep, whose xo-websock build
  registers both `utest.websock` and `utest.websock.live`
  (`ctest --test-dir xo-websock/.build -N`). So both compilers ran it, and
  the step passed. INFERRED from the workflow + the local equivalent: the
  logs of 576/577 could not be read -- no file under
  `/var/lib/forgejo/actions_log` (only 578's), the API
  (`actions/jobs/<id>/logs`) returns 404, no `sqlite3` on vpn1 to ask the
  db.
- **nix pipeline: does NOT run it.** The xo-websock nix package runs no
  tests at all (`No tests were found!!!` -- `pkgs/xo-websock.nix` passes no
  `-DENABLE_TESTING=1`; the doCheck gap, noted in issue 11). Its smoke-test
  passing says nothing about this test.

## Done when

- a test starts a `Webserver`, connects a real websocket client, and covers
  cases 1-5 above
- it runs in CI (both pipelines), or the ticket says why not
- `xo-build --sweep` ok in both stages
