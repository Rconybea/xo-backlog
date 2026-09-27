# 10 — an example that shows its own websocket state in the browser

Status: open
Type: feature / example
Raised: 2026-09-27

An example program that serves a page, and over websocket reveals its own
server-side state -- in particular how the websocket classes fit together --
drawn in the browser with d3 + svg.

Also a rehearsal for `.xo-backlog/xo-websock/issues/03` (browser-stepped
flywheel): the same server <-> page loop and JS protocol client, with no
arena/reactor machinery.

## What exists (measured 2026-09-27)

- **Static files.** `WebserverImplWsThread::init_mount_static`
  (`xo-websock/src/websock/Webserver.cpp`) mounts `/` from `./mount-origin`,
  default `index.html` -- relative to the process's working directory,
  hard-coded. The kalman-era demo copied its files into the build directory
  (`xo-websock/utest/CMakeLists.txt`, disabled stanza).
- **Structured state to the page.** A stream endpoint's sink renders any
  reflected type via PrintJson (`WebsocketSink::notify_ev_tp`). Reflection
  covers structs and vectors:
  `ls xo-reflect/include/xo/reflect/` -> `struct/`, `vector/`,
  `StructReflector.hpp`; used e.g. `xo-process/src/process/UpxEvent.cpp`.
- **Page -> server.** `StreamReceiver` (issue 05) handles
  `{"cmd": "send", "sub_id": N, "msg": ...}`; its reply goes back on that
  subscription.
- **Protocol** settled by issues 04/06/07: subscribe/subscribed, envelope
  `{"stream", "sub_id", "seq", "event"}`, unsubscribe/unsubscribed (+reason).
- **Live testing** harness: `WsTestClient`, `utest.websock.live` (issue 09).

## The gap: nothing exposes the server's internals

No API lists the state an introspection page needs:

| component | state | where |
|---|---|---|
| `UrlRouter` | registered http / stream endpoints | `http_map_`, `stream_map_` |
| `WsSessionTable` | live sessions by id | `session_map_` |
| `WsSessionRouter` | a session's subscriptions | `subscription_v_` |
| `WsSessionSender` | open / closed | `open_` |
| `WebserverImpl` | endpoints unregistered, subscriptions not yet ended | `removed_endpoint_v_` |
| (all `rp<>`-held) | reference counts | `ref::Refcount::reference_counter()` |

## Proposal (decided 2026-09-27, RC: pull first)

**Step 1 -- snapshot API.** `Webserver::snapshot()` returns a reflected value
type (e.g. `WebserverSnapshot`), gathered under each component's own lock:

- server: state, listen port
- endpoints (http and stream): stem, pattern, kind, identity, refcount --
  including removed-but-still-referenced ones
- sessions: id, sender identity + open + refcount, and per subscription:
  sub_id, stream name, endpoint identity, sink identity

Identities (e.g. object addresses) are what make the relationships drawable:
one `WsSessionSender` shared by a router and all its sinks; one
`DynamicEndpoint` shared by subscriptions across sessions; an unregistered
endpoint kept alive by a subscription it still serves.

Needs read-only listing methods on `UrlRouter` and `WsSessionRouter` (and the
session table's `for_each`). Unit-tested without a socket where possible;
`utest.websock.live` for the assembled snapshot.

**Step 2 -- the example.** `xo-websock/example/introspect/`, under
`XO_ENABLE_EXAMPLES`: registers a stream endpoint (e.g. `/introspect`) whose
`StreamReceiver` answers `{"cmd": "send", "msg": "refresh"}` with a snapshot
frame, plus a demo stream or two so there is something to look at. Page:
html + js (d3 + svg) that subscribes, requests refreshes (button / timer),
and draws the object graph.

**Later -- push.** Change hooks (session open/close, subscribe/unsubscribe,
register/unregister) so the page redraws without asking. Deferred: the page's
own subscription changes the state it observes, so hooks need care.

## Plan: increments (RC, 2026-09-27)

Supersedes the two-step proposal above: start with an almost empty but
working example, then add detail one piece at a time.

1. **Almost-empty example** -- no library change. Snapshot = `{listen_port,
   state}` from the public API; page draws one box. d3 from CDN; page files
   copied into the build dir (RC).
2. registered endpoints (`UrlRouter` listing)
3. sessions (`WsSessionTable` listing)
4. subscriptions per session (`WsSessionRouter` listing)
5. identities + refcounts -> shared sender / shared endpoint drawn as edges
6. push instead of pull

## Progress

**Increment 1, 2026-09-27.** Umbrella `3c60fca4` (with the mount_origin fix
below).

- `xo-websock/example/introspect/introspect.cpp` -> `websock_ex_introspect
  [port]` (default 7681; 0 = OS-picked, printed). Registers `/introspect`
  whose `IntrospectReceiver` (a `StreamReceiver`) answers `"refresh"` with a
  reflected `IntrospectSnapshot {listen_port, state}`; anything else is an
  error reply (the router turns the exception into one). SIGINT/SIGTERM stop
  and join cleanly.
- `mount-origin/index.html`, `introspect.js`: d3 7.9.0 from cdnjs; connects
  with protocol `lws-minimal`, subscribes, Refresh button (and one refresh on
  subscribe), draws one svg box sized to its label, shows the raw frame.
- `xo-websock/example/CMakeLists.txt` (new) + `add_subdirectory(example)`;
  under `XO_ENABLE_EXAMPLES`; POST_BUILD copies `mount-origin` beside the
  executable (the server's origin is `./mount-origin`, cwd-relative).

Verified 2026-09-27: run on port 0 from the build dir; `curl` `GET /` -> 200
(index.html), `GET /introspect.js` -> 200; node 22's global `WebSocket`
(protocol `lws-minimal`) got `{"cmd":"subscribed",...,"sub_id":0}` then
`{"stream":"/introspect","sub_id":0,"seq":0,"event":{"_name_":"IntrospectSnapshot","listen_port":43421,"state":"running"}}`.
PrintJson adds `"_name_"`. SIGTERM -> clean stop. `xo-build --sweep
--with-examples` ok in both stages; umbrella 49/49.

Page verified in a real browser by RC, 2026-09-27: draws correctly. No
automated test for the example.

**Found in RC's first real run: started from anywhere but its build dir, the
server served lws's fallback page** (`<img src="/libwebsockets.org-logo.svg">
... no dynamic content for uri [/] from mountpoint`) -- the static origin was
the cwd-relative `./mount-origin`, and a missing file falls through to the
dynamic handler. Reproduced by starting it from the repo root. Fixed (RC:
origin config + startup check), same uncommitted change:

- `WebserverConfig::mount_origin()` (default `./mount-origin`, as before) and
  `with_mount_origin(dir) const` (returns a copy); `init_mount_static` uses
  it -- lws keeps the pointer, into `WebserverImpl::ws_config_`. Bound in
  xo-pywebsock (`mount_origin` property, `with_mount_origin(dir)`); checked
  from python against `.build/python`.
- the example finds `mount-origin/` beside its executable (`/proc/self/exe`,
  else `argv[0]`), and exits 1 with the path it looked for if `index.html` is
  missing.
- Verified: started from the repo root -> real `index.html`, `introspect.js`
  200, websocket refresh ok; binary copied to a dir without `mount-origin` ->
  `introspect: page not found: ".../mount-origin/index.html"`, rc 1.
  utest.websock 39/498, utest.websock.live 5/65, umbrella 49/49,
  `xo-build --sweep --with-examples` ok.

**Increment 2, 2026-09-27 -- registered endpoints.** Umbrella `4f41c8a3`.

- Library: new `xo-websock/include/xo/websock/EndpointInfo.hpp` -- plain value
  `EndpointInfo {kind, stem, uri_pattern}`, `endpoint_kind_descr()`, and
  `EndpointKind` MOVED here from `DynamicEndpoint.hpp` (which includes it), so
  `Webserver.hpp` needs no `DynamicEndpoint.hpp`. `UrlRouter::endpoints()`
  (copies under the lock; http then stream, each by stem -- the maps are
  unordered, so sorted for a stable listing) and `Webserver::endpoints()`,
  delegating.
- Tests: `url-router-lists-its-endpoints` (registered out of order, one stem
  in both maps; order; unregistered one drops out),
  `webserver-lists-its-endpoints` (idle server). Falsified with a compiling
  change (stems sorted descending): fails at the first order check.
- Example: snapshot gains `endpoints: [{kind, stem, pattern}]` (reflected
  `IntrospectEndpoint`, kind as text; `std::vector` of a reflected struct
  needs nothing extra). Demo endpoints: http `/hello/${name}` (answers
  `/dyn/hello/<name>` -- http endpoints live under the `/dyn` mount) and
  stream `/demo/${id}` (subscribable, no frames yet).
- Page: server in the middle, http endpoints left, stream endpoints right,
  one link each; links and boxes in fixed layer groups so a refresh cannot
  draw a line over a box. First layout (server left, both columns right)
  was misleading -- the link to a stream ran behind an http box -- caught in
  a headless-Chrome screenshot, fixed.
- **CMake bug from increment 1, fixed:** the page was copied by a POST_BUILD
  step, which runs only when the executable relinks -- a page-only edit
  never reached the build dir (found when the screenshot did not change).
  Now one copy rule per page file (`copy_if_different`, `DEPENDS` its
  source, `GLOB_RECURSE CONFIGURE_DEPENDS`) under an `ALL` target the
  executable depends on. Verified: a page-only edit propagates.

Verified: `curl /dyn/hello/roland` -> `<html>hello, roland</html>`; websocket
refresh returns all three endpoints in order; headless Chrome
(`google-chrome --headless=new --screenshot`) shows the layout above.
utest.websock 41/527, utest.websock.live 5/65, umbrella 49/49,
`xo-build --sweep --with-examples` ok.

**Increment 3, 2026-09-27 -- sessions.** Implemented, awaiting review and
commit in the umbrella.

- Library: new `xo-websock/include/xo/websock/SessionInfo.hpp` --
  `SessionInfo {session_id, sender_open, n_subscription}`.
  `Webserver::sessions()` (sorted by id) collects via a new `const`
  `WsSessionTable::for_each`, each record's `info()` taking its router's lock
  inside the table's (lock order table -> router; a router never calls out
  holding its lock).
- Tests: `session-table-const-for-each-reads-live-sessions` (unit);
  `live-sessions-lists-each-connection` (live: two clients, one subscribed;
  ids ascending in connection order, both open, counts 0 / 1; a closed
  session leaves the listing). Server-side bookkeeping runs on its own thread,
  so the live test polls with a deadline (`wait_until`). Falsified with a
  compiling change (`info()` reporting 0 subscriptions): fails at the count.
  5/5 runs.
- Example: snapshot gains `sessions: [{id, sender_open, n_subscription}]`;
  page draws them in a row under the endpoint columns, linked up to the
  server (dashed if the sender is closed).

Verified: headless Chrome with a node client held open on `/demo/1` shows
"session 1 · 1 sub" (node) and "session 2 · 1 sub" (the page itself, on
`/introspect`). Cosmetic, left: the link to session 1 crosses the "websocket
sessions" heading. utest.websock 42/528, utest.websock.live 6/82, umbrella
49/49, `xo-build --sweep --with-examples` ok.

## Open

- **Identity:** raw addresses (simple, exactly what they are) or stable
  per-kind ids (nicer to read, needs a table).
- ~~**Static origin**~~ -- both: files copied beside the executable, and
  `WebserverConfig::with_mount_origin` so it runs from any directory.
- **d3:** vendored (works offline) or CDN.
- **Snapshot consistency:** per-component locks give a snapshot that is
  consistent per component, not globally atomic. Probably fine for a
  debugging view; say so on the API.

## Done when

- `Webserver::snapshot()` exists, reflected, tested
- `xo-websock/example/introspect` builds under `XO_ENABLE_EXAMPLES`, serves
  its page, and the page draws endpoints, sessions, subscriptions, senders
  and sinks with their sharing, refreshed on request
- `xo-build --sweep` ok in both stages
