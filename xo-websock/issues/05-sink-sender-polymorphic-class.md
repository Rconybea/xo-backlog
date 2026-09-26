# 05 — replace std::function callbacks in the websocket session path with polymorphic classes

Status: open
Type: refactor
Raised: 2026-09-26, follow-up to `.xo-backlog/xo-websock/issues/04`

`WebsocketSink::SendFn` (`xo-websock/include/xo/websock/WebsocketSink.hpp:33`)
is `std::function<void (std::string text)>`. It was introduced in issue 04 so a
sink's finished messages could go somewhere other than a live webserver (the
unit tests).

**The objection is allocation.** A `std::function` cannot be given an
allocator, so a callable too big for its small-buffer storage goes on the heap,
with no way for the caller to say otherwise. The webserver's sink captures
`rp<Webserver>` plus a session id (`WebsocketSink::make`,
`xo-websock/src/websock/WebsocketSink.cpp:126`). Under libstdc++'s rule -- only
small, trivially copyable callables are stored inline -- that capture is
heap-allocated, since `rp<>` is not trivially copyable. That is inferred from
the library's rule, not measured.

## Proposal

An abstract sender class with a virtual send, replacing `SendFn`:

- the webserver-backed implementation holds the webserver and session id;
- the unit tests' recorder becomes a small subclass
  (`xo-websock/utest/WsSessionRouter.test.cpp` builds sinks through
  `WebsocketSink::make(send_fn, ...)` in two cases);
- `WebsocketSink::make(send_fn, pjson, stream)` takes the sender instead.

A concrete subclass can be allocated however the owner chooses, which is the
point.

Open:

- plain virtual class, or a fomo facet (`ASender` / `obj<ASender>`), as
  xo-reactor2 does for its event sinks. The facet version fits the direction of
  the codebase; the plain class is smaller.
- ownership: does the sink own its sender (`rp<>`, which means `Refcount`), or
  borrow one whose lifetime the session guarantees?

## Also in scope: the session router's view of the server

Widened 2026-09-26 (RC). `WsSessionRouter` reaches the server through three
`std::function`s -- `EndpointLookup`, `SinkFactory`, `ReplyFn`
(`xo-websock/include/xo/websock/WsSessionRouter.hpp`) -- supplied as lambdas
when `WebserverImpl::notify_ws_session_open` builds each
`WebsocketSessionRecd` (`xo-websock/src/websock/Webserver.cpp`).

Those three are ONE role with three operations -- find an endpoint, make a
sink, reply to this session -- so the natural replacement is a single abstract
interface (working name `WsSessionHost`), one object per session:

- the webserver's implementation holds `WebserverImpl *` and the session id;
- the unit tests' `Fixture` becomes one subclass instead of three lambdas;
- `WsSessionRouter`'s constructor takes the interface.

An honest caveat on the allocation argument HERE: the production lambdas
capture only `[this]` and `[this, new_id]` -- small and trivially copyable,
which libstdc++ stores inline without allocating. So for these three the
objection is less that they allocate today than that the choice is made
inside the standard library rather than by us, and that three independent
callables stand in for one interface. The test fixture's lambdas capture more.

## Not in scope, but the same objection applies

The remaining `std::function` aliases on the path:

```bash
grep -rn "std::function<" xo-websock/include xo-websock/src xo-webutil/include
```

| alias | where |
|---|---|
| `StreamSubscribeFn`, `StreamUnsubscribeFn`, `StreamReceiveFn` | `StreamEndpointDescr` (xo-webutil) -- one set per registered endpoint |
| `HttpEndpointFn` | `HttpEndpointDescr` (xo-webutil) |

These are created once per registered endpoint, at startup, not per
connection. Worth deciding whether this ticket sets the pattern for them.

Also: the sink itself is still `new WebsocketSinkImpl(...)`
(`WebsocketSink.cpp:145`), and every outbound message is a `std::string` built
through a `std::stringstream`. Removing the `std::function` removes one heap
allocation per subscription, not per message. If per-message allocation is the
real target, that is a separate and larger change.

## Done when

- `WebsocketSink::SendFn` is gone; sinks send through the sender class
- the webserver-backed and test senders are subclasses
- `WsSessionRouter` takes one session-host interface instead of three
  `std::function`s; `WebserverImpl` and the test fixture implement it
- `utest.websock` unchanged in what it covers, and green
- `xo-build --sweep` ok in both stages
