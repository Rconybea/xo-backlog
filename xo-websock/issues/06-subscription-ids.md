# 06 — give subscriptions unique ids, so control messages are unambiguous

Status: implemented 2026-09-26, awaiting review and commit in the umbrella
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

- carried on every outbound envelope, e.g. `"sub_id": <id>`, so the client can
  demultiplex;
- used to address control messages: `{"cmd": "send", "sub_id": <id>, "msg": ...}`.

### Consequence: per-subscription unsubscribe

There is no client-side unsubscribe command today. Only closing the session
unsubscribes (`WsSessionRouter::unsubscribe_all`, from
`notify_ws_session_close`):

```bash
grep -n '"unsubscribe"' xo-websock/src/websock/*.cpp    # empty
```

Ids are exactly what such a command would need
(`{"cmd": "unsubscribe", "sub_id": <id>}`). Worth doing together, since it is
the first control message whose target must be unambiguous.

## Design (decided 2026-09-26)

**Server-assigned ids, for now.** Client-assigned ids are the eventual second
mode ("support both"); not in this ticket.

**The id is the subscription's index in `WsSessionRouter::subscription_v_`**,
and `WsSessionRouter::Subscription` gains an `id_`. Consequences:

- **Unsubscribe leaves a hole; slots are never compacted or reused.** Erasing
  would shift every later id. Reusing a freed slot would let a page still
  holding the old id steer a `send` into a different subscription -- a
  stale-id bug. So unsubscribe nulls the slot, and ids only grow within a
  session. The vector is bounded by subscribes over one session's life, so no
  reclamation is needed. Same shape as the flywheel's root set, without the
  free list.
- **The sink knows its id.** It must put `"sub_id"` on every envelope and is
  created before the endpoint's subscribe runs, so the id is passed in:
  `WsSessionRouter::SinkFactory` and `WebsocketSink::make` gain it.

**Protocol:**

```json
client  {"cmd": "subscribe",   "stream": "/flywheel"}
server  {"cmd": "subscribed",  "sub_id": 0, "stream": "/flywheel"}
server  {"stream": "/flywheel", "sub_id": 0, "seq": 0, "event": ...}
client  {"cmd": "send",        "sub_id": 0, "msg": ...}
client  {"cmd": "unsubscribe", "sub_id": 0}
```

- **The subscribe response goes out BEFORE the endpoint's subscribe function
  runs.** That function may send an initial frame immediately -- the flywheel
  demo's will -- and the page must learn its id before any frame tagged with
  it arrives. Order: assign id, reply `subscribed`, then call subscribe.
- **A failed subscribe gets an error reply** (`{"error": ..., "stream": S}`
  for an unknown stream). This retires issue 04's deliberately silent
  unmatched subscribe; the test
  `subscribe-to-an-unknown-stream-stays-silent` flips on purpose.
- **`send` is addressed by `"sub_id"` ONLY.** Stream-name addressing is dropped,
  and with it issue 04's "earlier subscription wins" tiebreak. It can return
  alongside client-assigned ids if wanted.
- **`unsubscribe` by `"sub_id"`** is new -- the first control message whose
  target must be unambiguous. Errors: unknown id, already unsubscribed.
- ids are non-negative integers, named `"sub_id"` EVERYWHERE -- envelope,
  `subscribed` reply, `send`, `unsubscribe` (RC, 2026-09-26). One name for one
  thing, so a page echoes back exactly the key it read off a frame.

## Open (superseded by the Design above, kept for the reasoning)


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

## What landed, 2026-09-26

As designed. Touches only xo-websock: `WsSessionRouter` (the protocol),
`WebsocketSink` (both `make()`s gain `sub_id`, envelope gains `"sub_id"`),
`Webserver.cpp` (the sink factory passes the id through). The `SinkFactory`
signature gains the id. No other subsystem calls `WebsocketSink::make`.

Replies and errors, as implemented:

| message | reply |
|---|---|
| subscribe, known stream | `{"cmd": "subscribed", "stream": S, "sub_id": N}` BEFORE the endpoint's subscribe runs |
| subscribe, unknown stream | `{"error": "unknown stream", "stream": S}` |
| subscribe, no stream | `{"error": "subscribe requires a \"stream\""}` |
| send / unsubscribe, missing or non-uint `sub_id` | `{"error": "<cmd> requires a \"sub_id\""}` |
| send / unsubscribe, never-assigned id | `{"error": "unknown sub_id", "sub_id": N}` |
| send / unsubscribe, retired id | `{"error": "already unsubscribed", "sub_id": N}` |
| send, endpoint without receive | `{"error": "stream does not accept messages", "stream": S, "sub_id": N}` |
| unsubscribe, active id | `{"cmd": "unsubscribed", "sub_id": N}` |

`sub_id` must pass jsoncpp's `isUInt()`, so a negative, fractional or string
id is rejected before `asUInt()` could throw.

### Calls taken without RC, for review

- **`unsubscribe` gets a reply**, `{"cmd": "unsubscribed", "sub_id": N}`.
  The design did not specify one. Frames already queued may still arrive after
  the page asks to unsubscribe, so the page needs a marker meaning "no more
  frames on this id". Trivially removable.
- **"unknown" and "already unsubscribed" are distinct errors**: an id never
  assigned, versus one retired. It costs nothing -- the slot says which -- and
  a page debugging a stale id wants to know that it WAS valid once.
- `n_subscription()` now counts ACTIVE subscriptions, excluding retired slots.

### Tests

`utest.websock`: 14 cases, 155 assertions (was 11 / 72). New: the subscribed
reply and its id; unknown stream is an error (issue 04's silent case, flipped
on purpose and renamed); **the subscribed reply precedes an initial frame sent
from the endpoint's subscribe function**, with replies and frames on one
ordered wire; one stream subscribed twice and addressed separately;
unsubscribe by id, including the retired-id errors and no double unsubscribe;
a retired id never reused; the envelope carrying `sub_id`. Every existing case
moved to `sub_id` addressing.

Six behaviours falsified, each with a change that compiles, each failing at
its intended assertion: reply sent after the endpoint's subscribe; slots
erased so ids are reused; send ignoring `sub_id`; envelope without `sub_id`;
unsubscribe leaving the slot live; unknown stream silent.

Umbrella ctest 48/48. `xo-build --sweep` ok in both stages:

```
stage 1: 73 attempted: 73 ok, 0 with no tests, 0 failed, 0 skipped
stage 2: 73 attempted: 47 ok, 26 with no tests, 0 failed, 0 skipped
```

## Done when

- every subscription has a server-assigned id -- its index in
  `subscription_v_`, never reused within a session -- carried on every
  outbound envelope as `"sub_id"`
- subscribe answers `subscribed` with the id BEFORE the endpoint's subscribe
  runs (a test pins the ordering against an endpoint that sends an initial
  frame), and answers an unknown stream with an error
- `send` addresses by `"sub_id"` only, unambiguously when one stream is
  subscribed twice
- a client can unsubscribe one subscription by id; a stale id is an error,
  never a different subscription
- tests cover two subscriptions to one stream, each addressed separately
- `xo-build --sweep` ok in both stages
