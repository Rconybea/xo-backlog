# 04 — inbound messages on a subscribed stream; sequence numbers in the envelope

Status: open
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
