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

## Done when

- a test starts a `Webserver`, connects a real websocket client, and covers
  cases 1-5 above
- it runs in CI (both pipelines), or the ticket says why not
- `xo-build --sweep` ok in both stages
