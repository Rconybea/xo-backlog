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

## Open

- **Identity:** raw addresses (simple, exactly what they are) or stable
  per-kind ids (nicer to read, needs a table).
- **Static origin:** copy page files into the build dir (as the old demo), or
  add an origin path to `WebserverConfig`.
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
