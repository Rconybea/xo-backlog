# 07 — UrlRouter: the endpoint maps as their own class; endpoints added and removed at runtime

Status: open
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
  source. Replacing or removing an endpoint then leaves existing subscriptions
  on the old one, alive, while new subscribes get the new one (or none).
  Subscribe and unsubscribe always pair on one endpoint's functions.

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
  and stream kept separate, unregister, and replace-while-subscribed.

## Open

- **What removal does to live subscriptions.** With refcounting, by default
  they keep the old endpoint alive and keep RECEIVING FRAMES until each client
  unsubscribes, since their sinks are still attached to the source. The
  alternative: removal ends them -- detach, and tell each client, e.g.
  `{"cmd": "unsubscribed", "sub_id": N, "reason": "endpoint removed"}`. That
  is arguably what someone removing an endpoint from python expects. But it
  needs an endpoint to know its subscribers, which today only each session's
  `WsSessionRouter` knows.
- **Replace vs reject** on registering an existing stem. With unregister
  available, rejecting a duplicate and requiring an explicit unregister first
  is the stricter contract; replacing silently is today's behaviour.
- `UrlRouter` sits beside `WsSessionRouter`: two routers, different jobs (URL
  to endpoint, server-wide; one session's commands to its subscriptions). If
  that grates, the second is the one to rename.

## Done when

- `UrlRouter` in its own `.hpp`/`.cpp`, holding both maps and the matching;
  `WebserverImpl` delegates
- `DynamicEndpoint` is reference-counted; maps and subscriptions hold `rp<>`
- endpoints can be registered and unregistered on a running server, from C++
  and from python
- `WsSessionRouter` takes `UrlRouter const &`; its tests use real routing
- `UrlRouter` has its own tests, including replace-while-subscribed (the old
  endpoint's unsubscribe still runs)
- the removal-vs-live-subscriptions question decided and tested
- `xo-build --sweep` ok in both stages
