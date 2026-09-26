# 06 — give subscriptions unique ids, so control messages are unambiguous

Status: open
Type: feature
Raised: 2026-09-26, follow-up to `.xo-backlog/xo-websock/issues/04`

Issue 04 addresses a control message to a subscription by STREAM NAME:

```json
{"cmd": "send", "stream": "/flywheel", "msg": ...}
```

A session may subscribe to the same stream more than once. Nothing prevents
it, just as nothing did before issue 04. When it does, the name is ambiguous.
Issue 04 resolves that by rule -- the message goes to the EARLIER subscription
(`WsSessionRouter::send`, `xo-websock/src/websock/WsSessionRouter.cpp`) -- which
is a tiebreak, not an answer: the client cannot address the later one at all.

Outbound messages have the same gap. The envelope is
`{"stream": ..., "seq": ..., "event": ...}`, so two subscriptions to one stream
produce indistinguishable messages, each with its own `seq` sequence
interleaved on one socket.

## Proposal

Each subscription gets an id, unique within its session:

- carried on every outbound envelope, e.g. `"sub": <id>`, so the client can
  demultiplex;
- used to address control messages: `{"cmd": "send", "sub": <id>, "msg": ...}`.

### Consequence: per-subscription unsubscribe

There is no client-side unsubscribe command today. Only closing the session
unsubscribes (`WsSessionRouter::unsubscribe_all`, from
`notify_ws_session_close`):

```bash
grep -n '"unsubscribe"' xo-websock/src/websock/*.cpp    # empty
```

Ids are exactly what such a command would need
(`{"cmd": "unsubscribe", "sub": <id>}`). Worth doing together, since it is
the first control message whose target must be unambiguous.

## Open

- **Who assigns the id.** Server-assigned needs an acknowledgement, since
  subscribe replies with nothing today; the client must wait for it before it
  can address the subscription. Client-assigned (the client puts an id in its
  subscribe request, the server echoes it on every envelope) needs no ack and
  no round trip. It is the common pattern for request/response over one
  socket, but the server must then reject a duplicate id within the session.
- **Whether addressing by stream name survives** alongside ids -- convenient
  for the common single-subscription case, but it keeps the tiebreak rule
  alive.
- **A subscribe acknowledgement** is worth having either way: the silent
  unmatched subscribe (kept deliberately in issue 04, and pinned by
  `subscribe-to-an-unknown-stream-stays-silent`) is the same "page gets
  nothing and nothing says why" problem, and an ack is the natural place to
  report it.

## Tests

`xo-websock/utest/WsSessionRouter.test.cpp` pins the current stream-name
behaviour. These change deliberately:

- `send-routes-to-the-named-stream-only` -- the addressing it pins changes;
- `subscribe-to-an-unknown-stream-stays-silent` -- if an ack lands.

## Done when

- every subscription has an id unique within its session, carried on every
  outbound envelope
- `send` can address a subscription by id, unambiguously when one stream is
  subscribed twice
- a client can unsubscribe one subscription by id
- tests cover two subscriptions to one stream, each addressed separately
- `xo-build --sweep` ok in both stages
