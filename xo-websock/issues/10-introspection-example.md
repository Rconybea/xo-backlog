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
