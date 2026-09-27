# 07 — UrlRouter: the endpoint maps as their own class; endpoints added and removed at runtime

Status: done 2026-09-26, umbrella `a19de4bd` (last step)
Type: refactor / feature
Raised: 2026-09-26, from `.xo-backlog/xo-websock/issues/05`

`WebserverImpl` holds two endpoint maps and the lookup logic over them. Pull
exactly those out into a class, `UrlRouter` -- "router" as the term of art for
mapping URLs to the code that acts on them -- with its own `.hpp`/`.cpp`. The
session router and the no-socket unit tests then use the REAL routing,
prefix-matching included, instead of a fake.

## What moves

Everything that touches the maps, in `xo-websock/src/websock/Webserver.cpp`:

```bash
grep -n "stem_map_\|stream_map_\|lookup_pattern\|lookup_stem\|EndpointMap" \
     xo-websock/src/websock/Webserver.cpp
```

| site | what |
|---|---|
| members `stem_map_`, `stream_map_` | http and stream endpoints, keyed by stem ("longest non-variable URI prefix") |
| `register_http_endpoint`, `register_stream_endpoint` | insert |
| session-open lambda | stream lookup, for each session's `WsSessionRouter` |
| `dynamic_http_response` | http lookup |
| `lookup_stem`, `lookup_pattern` | the shared matching: whole uri, then shorter prefixes ending in `/`, then shorter prefixes |

About 110 lines, moving as a unit. Sketch:

```cpp
/* server-wide: which endpoint serves a uri or stream name */
class UrlRouter {
public:
    void register_http(HttpEndpointDescr const & descr);
    void register_stream(StreamEndpointDescr const & descr);
    void unregister_http(std::string const & uri_pattern);
    void unregister_stream(std::string const & uri_pattern);

    /* longest-stem match, as lookup_pattern does today; null if none */
    rp<DynamicEndpoint> find_http(std::string const & uri) const;
    rp<DynamicEndpoint> find_stream(std::string const & stream_name) const;

private:
    mutable std::mutex mutex_;
    EndpointMap http_map_;     /* was stem_map_ */
    EndpointMap stream_map_;
};
```

## Ownership: endpoints come and go at runtime (decided 2026-09-26, RC)

Endpoints can be added and removed on demand. They will be bound to python,
and XO cannot impose lifetime rules on arbitrary python code. So:

- **`DynamicEndpoint` becomes reference-counted.** `UrlRouter`'s maps hold
  `rp<DynamicEndpoint>`; `make_http` / `make_stream` return `rp<>`, not
  `unique_ptr`.
- **Each subscription holds its endpoint by `rp<>`.** A subscription must
  later call the SAME endpoint's unsubscribe, to detach its sink from the
  source. Holding it by `rp<>` means that endpoint stays alive for as long as
  the subscription needs it -- in particular across an unregister, until the
  subscription has been ended (see below) -- so subscribe and unsubscribe
  always pair on one endpoint's functions.

  Today a subscription holds a raw `DynamicEndpoint *`, and re-registering a
  stem frees the old endpoint out from under it
  (`this->stream_map_[stem] = std::move(endpoint)`). This fixes that hazard,
  which was noted in issue 04.
- **`UrlRouter` locks its maps.** Registration happens on the application's
  thread, and lookup on the webserver's service thread. The maps are
  unsynchronized today, so registering after `start_webserver()` is a data
  race today. `find_*` copies the `rp<>` out under the lock, and the caller
  uses the endpoint with the lock released -- endpoint functions may send,
  which re-enters the server.
- **Unregistration is new** (`unregister_http` / `unregister_stream`); nothing
  can remove an endpoint today. `Webserver` gains matching methods, bound in
  xo-pywebsock alongside `register_*_endpoint`.

Worth knowing: a kept-alive endpoint keeps its captures alive. xo-reactor2websock's
stream endpoint captures `rp<AbstractSource>`, so an old endpoint held by a
lingering subscription keeps its source alive too. That is correct, but not
obvious.

### Relation to issue 05 (`WsSender`)

A reference cycle exists today: server -> endpoint -> subscribe fn -> source
-> attached sink -> the sink's send fn, which captures `rp<Webserver>` ->
server. It is broken whenever a subscription ends, and session close
unsubscribes everything, so it resolves in practice. Refcounted endpoints
lengthen that chain. Issue 05's `WsSender` holds a plain `WebserverImpl *`,
removing the edge back to the server. Landing 05 first, or with this, keeps
refcounted endpoints cycle-free.

## Removing an endpoint ends its live subscriptions (decided 2026-09-26, RC)

Without this, refcounting alone would leave existing subscriptions on a
removed endpoint alive -- and still RECEIVING FRAMES, since their sinks stay
attached to the source -- until each client unsubscribed. Instead, removal
ends them, and tells each client:

```json
{"cmd": "unsubscribed", "sub_id": N, "reason": "endpoint removed"}
```

This is issue 06's `unsubscribed` reply plus a `reason`. As with a
client-initiated unsubscribe, frames already queued may still arrive; the
reply marks the end.

Mechanism, proposed:

- **The server walks its sessions; no back-pointers.** Only each session's
  `WsSessionRouter` knows which of its subscriptions use a given endpoint. On
  unregister, each session's router ends the subscriptions whose endpoint is
  the removed one (`rp<>` identity), running that endpoint's unsubscribe
  function for each. There are few sessions, so the walk is cheap. The
  alternative, an endpoint keeping a list of its subscribers, adds endpoint ->
  session references, and with them the cycles issue 05's `WsSender` removes.
- **The forced unsubscribe runs on the webserver's service thread.**
  Unregister is typically called from python's thread, but subscriptions,
  sinks and replies belong to the service thread. So unregister removes the
  endpoint from `UrlRouter` at once (new subscribes fail immediately), queues
  the ending of live subscriptions, and wakes the service loop -- the way
  `WebsocketSessionRecd::send_text` does with `lws_cancel_service` -- rather
  than detaching sinks from the application thread while a source may be
  delivering into them.

Refcounting still matters under this rule. Between unregister and the queued
work running, live subscriptions hold the endpoint alive, and their
unsubscribe still runs on the endpoint that subscribed them.

## A duplicate registration is rejected (decided 2026-09-26, RC)

Registering an endpoint whose stem is already present is an ERROR. Replacing
one takes an explicit unregister first. Today a duplicate silently replaces
the existing endpoint (`this->stream_map_[stem] = std::move(endpoint)`).

- **Proposed: `register_*` throws** on a duplicate, so a python caller gets an
  exception rather than a return value it can ignore.
- **"Duplicate" means same STEM, not same pattern.** The maps are keyed by
  stem, the longest literal prefix. `/fw/${a}` and `/fw/${a}/detail` share the
  stem `/fw/`, so the second is rejected even though the patterns differ. It
  was silently replaced before, so rejecting makes an existing limit of the
  keying visible rather than creating it. Keying by full pattern would lift
  it, but that is a change to matching, not in scope here.

## Progress

**Step 1, 2026-09-26 -- `DynamicEndpoint` reference-counted** (RC's suggestion:
land it alone, ahead of the rest of 05/07). Umbrella `e94f5695`.

- `DynamicEndpoint : public ref::Refcount`; `make_http` / `make_stream` return
  `rp<DynamicEndpoint>`
- `WebserverImpl`'s `EndpointMap` holds `rp<>`; lookups still return a raw
  pointer (`UrlRouter::find_*` returning `rp<>` is the later step)
- `WsSessionRouter::Subscription::endpoint_` is `rp<DynamicEndpoint>`

This alone fixes the re-registration hazard: the old endpoint lives until its
last subscription ends, and unsubscribe runs on the endpoint that subscribed.
Pinned by `an-endpoint-replaced-while-subscribed-outlives-the-map`
(`xo-websock/utest/WsSessionRouter.test.cpp`), which tracks the old endpoint's
lifetime through a `shared_ptr` its callbacks capture. Falsified: a raw-pointer
subscription fails that test at the liveness check, before the unsubscribe
that would have called into freed memory.

utest.websock 15 cases / 160 assertions; umbrella 48/48; `xo-build --sweep` ok
in both stages (73/73; 47 ok, 26 no tests).

**Step 2, 2026-09-26 -- `UrlRouter` extracted** (RC: extraction only).
Umbrella `13049348`.

- `xo-websock/include/xo/websock/UrlRouter.hpp`, `src/websock/UrlRouter.cpp`:
  `register_http` / `register_stream`, `find_http` / `find_stream` returning
  `rp<DynamicEndpoint>`, one mutex over both maps. `stem_map_` renamed
  `http_map_`. `lookup_stem` / `lookup_pattern` moved verbatim.
- `WebserverImpl` holds a `UrlRouter` and delegates. The session-open lambda
  still hands `WsSessionRouter` a raw pointer (`EndpointLookup` unchanged);
  switching `WsSessionRouter` to `UrlRouter const &` is the next step (RC).
- Behaviour unchanged: a duplicate stem still replaces. Rejecting it waits for
  unregister, else nothing could replace an endpoint at all.
- New `xo-websock/utest/UrlRouter.test.cpp`, 8 cases: empty, literal whole-uri,
  `${var}` stored under its stem, longest stem wins (either registration
  order), a `/`-terminated stem beats a longer bare prefix (pinned: the
  matching is NOT strictly longest-prefix), bare-prefix fallback, http and
  stream kept apart (one stem in both maps), re-register replaces while a held
  `rp<>` keeps the old endpoint usable. Falsified twice with compiling
  changes: `find_http` searching the stream map; skipping the whole-uri step.

utest.websock 23 cases / 187 assertions; umbrella 48/48; `xo-build --sweep`
ok in both stages (73/73; 47 ok, 26 no tests).

**Step 3, 2026-09-26 -- `WsSessionRouter` takes `UrlRouter const &`.**
Umbrella `5a6b827a` (with two `Webserver.cpp` cleanups: an undefined, unused
`lookup_dynamic_http_stem` declaration and an unused `using Alist`).

- `EndpointLookup` is gone; the router holds `UrlRouter const & url_router_`
  (a reference, RC) and `subscribe()` stores `find_stream()`'s `rp<>` directly,
  closing the raw-pointer window between lookup and the subscription taking
  ownership.
- Borrowed, so the `UrlRouter` must outlive every session's router:
  `WebserverImpl::url_router_` is declared before `session_v_`, so sessions are
  destroyed first. Commented at both ends.
- `xo-websock/utest/WsSessionRouter.test.cpp`: the fixture's exact-match
  `std::map` is replaced by a real `UrlRouter`; every case registers through
  `register_stream(StreamEndpointDescr(...))`. New case
  `subscribe-resolves-a-stream-name-through-its-pattern` (`/fw/${id}` serves
  `/fw/7` and `/fw/8`; `/other/7` is an error). Falsified with a compiling
  exact-match-only `find_in`: fails at its first `n_subscription` check.

utest.websock 24 cases / 199 assertions; umbrella 48/48; `xo-build --sweep`
ok in both stages (73/73; 47 ok, 26 no tests).

**Step 4, 2026-09-26 -- unregister + duplicate rejection, in `UrlRouter`
only** (RC: bounded; no Webserver/python API yet). Umbrella `1ba706e8`.

- `register_http` / `register_stream` throw `std::runtime_error` on a stem
  already in that map; the message names both patterns and says to unregister
  first. Propagates through `Webserver::register_*_endpoint` (documented in
  `Webserver.hpp`); no in-tree caller registers a duplicate.
- `bool unregister_http(uri_pattern)` / `unregister_stream(uri_pattern)`.
  Calls taken without RC, for review:
  - **exact pattern required**: `/fw/${b}` does not remove a registered
    `/fw/${a}`, though they share a stem. Needed `DynamicEndpoint::uri_pattern()`.
  - **not found returns false, does not throw** -- removal stays idempotent.
    Arguable against the "python can ignore a return value" reasoning for
    register; flip if preferred.
  - the removed `rp<>` is released after the lock, so an endpoint dtor (and its
    captures' dtors) never runs under the router's mutex.
- Live subscriptions on an unregistered endpoint are NOT ended: they keep it
  alive via `rp<>` and keep receiving frames. Unreachable until the
  Webserver/python unregister API exists (steps below).
- Tests: `url-router-reregistering-a-stem-replaces` flipped to
  `url-router-rejects-a-duplicate-stem`; new `unregister-then-register-replaces`
  and `unregister-needs-the-exact-pattern`. The router's
  `an-endpoint-replaced-while-subscribed-outlives-the-map` now unregisters
  before re-registering. Falsified with compiling changes: duplicate check
  disabled; pattern comparison in unregister disabled -- each fails at its
  intended assertion.

utest.websock 26 cases / 215 assertions; umbrella 48/48; `xo-build --sweep`
ok in both stages.

**Step 5, 2026-09-26 -- the router operation** (RC). Umbrella `137e7bb6`. (Issue 05 landed first, as planned: the
session's sender holds a plain `WebserverImpl *`, so no cycle.)

- `std::size_t WsSessionRouter::end_subscriptions_on(rp<DynamicEndpoint> const &)`:
  under the lock, retires every ACTIVE slot whose endpoint is the given one
  (by `rp<>` identity, not stem); then, lock released, in sub_id order, runs
  that endpoint's unsubscribe and replies
  `{"cmd": "unsubscribed", "sub_id": N, "reason": "endpoint removed"}`.
  Returns the count. Service thread, like `perform_cmd`.
- Tests (`[removal]` in `xo-websock/utest/WsSessionRouter.test.cpp`, 3 cases):
  ends only the removed endpoint's subscriptions (another stream's survives
  and still receives), replies with the reason in sub_id order, ended ids
  retired, a second call ends nothing; an already-unsubscribed slot is not
  unsubscribed again; an old endpoint replaced at the same stem ends only its
  own. Falsified with compiling changes: match by stem (the replacement case
  fails); slots not retired (3 cases fail).

utest.websock 37 cases / 489 assertions; umbrella 48/48; `xo-build --sweep`
ok in both stages.

**Step 6, 2026-09-26 -- Webserver + python unregister API** (RC).
Umbrella `a19de4bd`.

- `UrlRouter::unregister_http` / `unregister_stream` now return the removed
  `rp<DynamicEndpoint>` (null if none) instead of `bool`; the webserver needs
  the endpoint to end its subscriptions. Existing tests unchanged (`rp<>` tests
  as bool).
- `Webserver::unregister_http_endpoint` / `unregister_stream_endpoint`
  (`bool`), documented in `Webserver.hpp`. http: just `UrlRouter`. stream:
  out of `UrlRouter` at once (new subscribes fail), then pushed on
  `WebserverImpl::removed_endpoint_v_` (own mutex) and the service thread
  woken with `lws_cancel_service`, as `interrupt_stop_webserver` does.
- `WebserverImpl::lws_end_removed_subscriptions`, run from
  `LWS_CALLBACK_EVENT_WAIT_CANCELLED` before `lws_write_pending_traffic`:
  swaps out the queue, collects routers under the session table's lock, then
  calls `end_subscriptions_on` with it RELEASED (replies re-enter
  `send_text` -> the table lock, not recursive).
- xo-pywebsock: `Webserver.unregister_http_endpoint(uri_pattern)`,
  `unregister_stream_endpoint(uri_pattern)`. Checked by hand against the
  module the umbrella builds, `.build/python/xo/websock.*.so` (import
  `xo.websock`): methods present, docstrings, `False` for an unknown pattern.
  (A stale `.build/xo-pywebsock/src/pywebsock/xo_pywebsock.*.so` from
  2026-09-13 lingers from an earlier layout -- importing THAT shows none of
  this.)
- New `xo-websock/utest/Webserver.test.cpp`, 2 cases, on a server made but
  never started: exact-pattern unregister, true/false results, duplicate
  rejected until unregistered, http and stream removed separately.
- NOT tested: the live path -- queue, wakeup, drain, the `unsubscribed`
  reaching a real client. Needs a started server, a websocket client, and a
  stream endpoint; none in `utest.websock`, and no client (python websockets,
  websocat) in this environment. The router half is covered by step 5.

utest.websock 39 cases / 498 assertions; umbrella 48/48; `xo-build --sweep`
ok in both stages.

**Closed 2026-09-26 (RC)** with one "Done when" item met only below the
socket: unregistering ends live subscriptions is tested at the router
(`[removal]`, step 5), not on a running server. The live end-to-end test is
`.xo-backlog/xo-websock/issues/09`.

## Consequences

- **`WsSessionRouter` takes `UrlRouter const &`** in place of its
  `EndpointLookup` function. Endpoints are server-wide and outlive every
  session. This retires issue 05's proposed `WsSessionHost` altogether: with
  `WsSender` absorbing `ReplyFn` and `SinkFactory`, the endpoint lookup was the
  only piece left, and it was never per-session.
- **Tests use the real routing.** `xo-websock/utest/WsSessionRouter.test.cpp`
  registers through `register_stream()` instead of an exact-match map.
  `lookup_pattern` has no tests at all today; `UrlRouter` gets its own:
  longest-stem wins, `${var}` patterns, the `/`-then-no-`/` prefix order, http
  and stream kept separate, a duplicate stem rejected, and
  unregister-while-subscribed (the old endpoint's unsubscribe still runs).

## Open

- `UrlRouter` sits beside `WsSessionRouter`: two routers, different jobs (URL
  to endpoint, server-wide; one session's commands to its subscriptions). If
  that grates, the second is the one to rename.

## Done when

- `UrlRouter` in its own `.hpp`/`.cpp`, holding both maps and the matching;
  `WebserverImpl` delegates
- `DynamicEndpoint` is reference-counted; maps and subscriptions hold `rp<>`
- endpoints can be registered and unregistered on a running server, from C++
  and from python
- ~~`WsSessionRouter` takes `UrlRouter const &`; its tests use real routing~~ step 3
- `UrlRouter` has its own tests: registering a duplicate stem throws; unregister
  then re-register succeeds
- unregistering an endpoint ends every live subscription on it, in every
  session, on the service thread; each client gets `unsubscribed` with
  `"reason": "endpoint removed"`, and the endpoint's unsubscribe runs once per
  subscription -- tested
- `xo-build --sweep` ok in both stages
