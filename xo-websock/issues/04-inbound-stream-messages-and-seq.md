# 04 — inbound messages on a subscribed stream; sequence numbers in the envelope

Status: implemented 2026-09-26, awaiting review and commit in the umbrella
Type: feature
Raised: 2026-09-26, for `.xo-backlog/xo-websock/issues/03` (browser-stepped AllocFlywheel demo)

Two xo-websock features the flywheel demo needs, both generic, so every stream
gets them:

1. the browser can send an application message **on a stream it has
   subscribed to**, and the stream's endpoint handles it;
2. every outbound message carries a per-subscription **sequence number**.

Both keep to the principle from issue 02: xo-websock carries frames and never
shapes them.

## Why

Issue 03 decided the demo is a C++ example the BROWSER steps: a "next" control
sends a command, the demo performs one mutation and sends one frame back.
Inbound messages today can only subscribe: `LWS_CALLBACK_RECEIVE` hands every
message to `WebserverImpl::perform_ws_cmd`
(`xo-websock/src/websock/Webserver.cpp:1850-1865`), which acts on
`{"cmd": "subscribe", ...}` and ignores everything else.

## Design (decided 2026-09-26)

### Messages addressed to a subscribed stream

```json
{"cmd": "send", "stream": "/flywheel", "msg": <any JSON value>}
```

`StreamEndpointDescr` gains an optional third function beside subscribe and
unsubscribe: a **receive** function. The server finds the sending session's
subscription to that stream and calls the receive function with the message
**and that subscription's `WebsocketSink`**. A handler replies with
`sink->notify_ev_tp(...)`, and the reply reaches exactly the session that asked
-- no session bookkeeping in application code.

A separate registry of command endpoints independent of streams was considered
and rejected. It is more general, but the handler gets a bare session id and
must find its own way back to the right socket. For "step, then show me", tying
the command to its stream is the better fit.

### `msg` is JSON, handed over parsed

Decided: JSON is what the browser speaks natively, so the http layer handles it
natively. The handler receives `msg` as jsoncpp's `Json::Value`.

Consequences, accepted:

- `<json/json.h>` enters xo-websock's PUBLIC headers. **Update
  `.xo-backlog/xo-cmake/issues/06`'s survey row** for xo-websock/jsoncpp from
  "private, consumers need nothing" to "headers public, no linkage" -- the same
  case as Eigen in xo-kalmanfilter. The `find_dependency(jsoncpp CONFIG)` added
  to `websockConfig.cmake.in` in issue 02 stays load-bearing.
- Replacing jsoncpp later (a hypothetical xo-native equivalent) becomes an API
  change for handlers rather than an internal one. An xo-owned view type over
  the parsed value would avoid that. Not built now: one consumer, and the view
  type would arrive with the replacement if it ever does.

### Handlers run on the webserver's service thread

This is a CONTRACT, to be documented where the receive function is declared.
It is what makes the demo race-free: step and frame happen on one thread,
inside `LWS_CALLBACK_RECEIVE`. The flip side: a handler must not block, because
the whole server stalls while it runs.

Replying from inside the callback is already anticipated.
`WebsocketSessionRecd::subscribe_endpoint` drops its session lock before
calling the endpoint's subscribe function precisely because "subscribe may in
principle call WebserverImpl.send_text()" (`Webserver.cpp:397`). The receive
path must do the same: no session lock held across the handler.

### Sequence number in the envelope

Outbound messages are already wrapped by `WebsocketSinkImpl::notify_ev_tp`
(`xo-websock/src/websock/WebsocketSink.cpp`) as `{"stream": ..., "event": ...}`.
Add `"seq"` from the sink's own count, `n_in_ev`, which is per subscription.

Decided: sequencing is a property of the TRANSPORT, not of the native
publisher. A model of a native data structure should not have to know it is
being streamed. This supersedes issue 03's note that a frame sequence number
would go in `JsonPrinter_AllocFlywheel`.

Backward compatible with the only existing client: the kalman-era page reads
`.stream` and `.event` and nothing else
(`xo-websock/utest/mount-origin/ex_websock.js:799-804`).

## Things the implementation must get right

- **Store the stream name on the subscription.** `WebsocketSubscriptionRecd`'s
  field `incoming_uri_` actually holds the ENTIRE subscribe command text --
  `Webserver.cpp:1415` passes `incoming_cmd` -- so there is nothing to match a
  `send` against today. Rename the field to what it holds, or add
  `stream_name_`.
- **Unknown stream, or not subscribed to it:** log and ignore, as an unmatched
  subscribe does today. Replying with an error message instead is an option.
  Undecided.
- **Parse failure:** today `perform_ws_cmd` logs the failure and carries on
  reading an empty `root`. Harmless for subscribe (nothing matches); `send`
  should stop at the failure instead.
- **Fragmented messages:** the receive case never checks
  `lws_is_final_fragment()`, so a message libwebsockets delivers in pieces
  would be processed piecewise. Commands this small always arrive whole. A
  comment at the receive site is enough for now.
- `seq` base (0 or 1) -- pick one and document it.

## Open: how to test it

No automated websocket test exists anywhere in the tree, only the two manual
browser pages in `xo-websock/utest/mount-origin/`:

```bash
grep -rlE "lws_client_connect|websockets\.connect|new WebSocket" \
     --include=*.cpp --include=*.py --include=*.js . | grep -v '\.build/'
```

Candidates, not yet weighed: route through `perform_ws_cmd` via a test seam
without a socket; a real `Webserver` on an ephemeral port with a libwebsockets
client in a C++ utest; or a python client (needs a websocket library in the
nix environment -- unverified whether one is there). The routing logic
(find subscription by stream, call receive with the right sink, `seq`
increments per subscription) is the part worth pinning, whatever the harness.

## What landed, 2026-09-26

Decisions taken after the design above: `seq` is 0-based (RC); a `send` to a
stream the session has not subscribed to gets an error reply (RC); tests use a
socket-free seam (RC chose harness A; a python client is a later follow-up).

| where | change |
|---|---|
| xo-webutil `StreamEndpointDescr` | `StreamReceiveFn`, an optional 4th ctor argument; `Json::Value` forward-declared, so no new dependency; threading contract documented there |
| xo-websock `DynamicEndpoint` | carries the receive fn; `has_receive()`, `receive()` |
| xo-websock **`WsSessionRouter`** (new) | one per session; owns the subscription list and all command handling; no libwebsockets. Reaches the server through 3 injected fns: find endpoint, make sink, reply to session |
| xo-websock `WebsocketSink` | second `make(send_fn, pjson, stream)`; the webserver's `make()` wraps it; envelope gains `"seq"` |
| xo-websock `Webserver.cpp` | `WebsocketSubscriptionRecd`, `subscribe_endpoint`, `readjson_` and the parsing in `perform_ws_cmd` removed -- the session record owns a router and delegates; 145 lines out, 78 in |

Error replies are `{"error": <reason>, "stream": <name>}`, with "stream"
absent when there is none to name. Reasons: `malformed json: ...`,
`message is not a json object`, `send requires a "stream"`,
`not subscribed to stream`, `stream does not accept messages`,
`stream handler failed: <what()>`.

**A server crash fixed along the way.** A client sending a json STRING or ARRAY
crashed the server: jsoncpp throws `Json::LogicError` from `operator[]` on a
non-object (and from `asString()` on a non-string), and the exception escaped
into libwebsockets' C callback. The router type-checks before any field
access, and catches a handler's `std::exception` and turns it into an error
reply. Both are pinned by tests.

### Calls taken without RC, for review

- **The unit tests live in `xo-websock/utest/`, next to the kalman demo.**
  `add_subdirectory(utest)` is now live. The demo's build stanza is kept
  byte-identical inside `if(FALSE)`, and none of its files were touched. That
  is why the tests' main is `websock_unit_main.cpp`: `websock_utest_main.cpp`
  is the demo's. Issue 01 records this; its delete-or-port decision is
  unaffected.
- **A `send` whose message fails to parse, or is not an object, gets an error
  reply**, not only the not-subscribed case. It is on the send path in spirit
  (the server cannot know it was meant to be a send), and silence here is the
  "page gets nothing and nothing says why" problem.
- **An unmatched `subscribe`, and an unknown `cmd`, stay silent**, as before --
  an explicit scope limit, pinned by a test so a later change is deliberate.
  Arguably the same problem as above.
- **Subscribing twice to one stream is allowed**, as before. A `send` goes to
  the EARLIER subscription.
- **`msg` is optional**: absent reads as json null, and the handler decides.

### Tests

`utest.websock`, 11 cases, 72 assertions -- the first automated tests
xo-websock has had. They cover: subscribe; silent unknown subscribe; send
reaching receive with the subscription's OWN sink; routing among two streams;
the four error cases; malformed and non-object input never throwing; a
throwing handler; unsubscribe_all; `seq` 0-based and per subscription; and a
handler's reply going out enveloped and sequenced on its own subscription only
(real sinks, no socket).

Each of seven behaviours was falsified with a change that compiles, and each
failed at its intended assertion: routing ignoring the stream name; 1-based
seq; no error when unsubscribed; object check removed; handler exceptions not
caught; receive handed a fresh sink instead of the subscription's; unsubscribe
skipped.

Umbrella ctest 48/48 (+1, utest.websock). `xo-build --sweep` ok in both stages:

```
stage 1: 73 attempted: 73 ok, 0 with no tests, 0 failed, 0 skipped
stage 2: 73 attempted: 47 ok, 26 with no tests, 0 failed, 0 skipped
```

xo-websock moved from "no tests" to "ok".

### Noticed, not acted on

- `WebsocketSinkImpl::notify_ev_tp` has debug logging hard-wired on
  (`XO_DEBUG_(true)`), so every outbound frame is logged. Visible in the test
  output, and the flywheel demo would log every frame too.
- A subscription holds a raw `DynamicEndpoint *`. Re-registering a stream
  replaces its `unique_ptr` in the webserver's stream map, which would leave
  existing subscriptions dangling. Pre-existing (the old record held the same
  raw pointer); harmless while endpoints are registered once, before
  `start_webserver`.

## Done when

- `{"cmd": "send", ...}` reaches the subscribed stream's receive function with
  the parsed `msg` and that subscription's sink; a reply from the handler
  reaches only that session
- outbound envelopes carry `"seq"`, per subscription, documented base
- the service-thread contract is documented at the receive function's
  declaration
- ticket 06's survey row updated as above
- the routing is covered by an automated test (harness per the open question)
- `xo-build --sweep` ok in both stages
