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

## Proposal: `WsSender` (sketched 2026-09-26; name chosen by RC)

**One `WsSender` per session, shared by that session's router (replies) and
every sink it creates (frames).** It is the layer BELOW `WebsocketSink`: the
sink turns an event into json (envelope, `sub_id`, `seq`); the sender puts
finished text on the wire. Hence not another "sink".

It replaces BOTH `WebsocketSink::SendFn` and `WsSessionRouter::ReplyFn`. The
two have the identical signature `void (std::string text)`, and in production
are the identical operation: `Webserver::send_text(session_id, text)` for one
session.

```cpp
namespace xo::web {
    /** @brief delivers finished outbound text to ONE websocket session.
     *
     *  THREADING: send_text() may be called from any thread; it must not
     *  block (the production sender queues).
     **/
    class WsSender : public ref::Displayable {
    public:
        /** send @p text as one complete websocket message.
         *  After the session closes, dropped rather than delivered.
         **/
        virtual void send_text(std::string text) = 0;

        /** false once the destination session has closed **/
        virtual bool is_open() const = 0;
    };
}
```

Production implementation, private to `Webserver.cpp`:

```cpp
class WsSessionSender : public WsSender {
public:
    WsSessionSender(WebserverImpl * websrv, uint32_t session_id);

    void send_text(std::string text) override;   // -> websrv_->send_text(session_id_, ...), if open
    bool is_open() const override;
    void close();                                 // from notify_ws_session_close

private:
    WebserverImpl * websrv_;
    uint32_t session_id_;
    std::atomic<bool> open_{true};
};
```

Wiring:

- `WebsocketSessionRecd` creates one `rp<WsSender>` at session open, and
  closes it at session close.
- `WsSessionRouter` takes that `rp<WsSender>` in place of `ReplyFn`.
- `WebsocketSink::make(rp<WsSender>, pjson, stream, sub_id)` replaces BOTH
  existing `make()` overloads. The webserver-specific one exists only to build
  a send function from `(websrv, session_id)`.
- The tests implement `RecordingSender : WsSender` holding a
  `std::vector<std::string>`; it replaces the recorder lambdas in
  `xo-websock/utest/WsSessionRouter.test.cpp`.

A concrete subclass can be allocated however the owner chooses, which is the
original point of the ticket.

### `close()` fixes a hazard in the current design

From reading the code; NOT reproduced. A sink today captures a bare
`session_id` (`WebsocketSink::make`, `xo-websock/src/websock/WebsocketSink.cpp`).
When a session closes its id goes on a free list, and the next connection
reuses it (`WebserverImpl::notify_ws_session_close` /
`notify_ws_session_open`, `xo-websock/src/websock/Webserver.cpp`). So a sink
the application RETAINS past its session's close would start writing into a
different client's session. The flywheel demo is exactly the kind of code that
might keep a sink for later pushes.

`close()` removes the hazard: the sink holds the session's sender rather than
its id, and a closed sender drops what it is given. Worth a test that
reproduces the misdelivery first.

### Consequence for `WsSessionHost` (below)

With the router holding a sender and `pjson`, it can make sinks itself via
`WebsocketSink::make(sender, pjson, stream, sub_id)`. `SinkFactory` then
disappears, and `ReplyFn` IS the sender. What remains is the endpoint lookup,
which was never per-session -- see the section below and issue 07.

### Open

1. **`std::string text` by value**, as sketched: the production sender moves it
   into its queue without copying. `std::string_view` would force a copy
   there. `std::string &&` is equivalent but stricter for callers.
2. ~~Ownership~~ **Decided 2026-09-26 (RC): `rp<WsSender>`.** The simplest way to
   share one sender between a router and sinks that may outlive it. It costs
   one `Refcount` allocation per SESSION, against one per subscription today
   with `SendFn`.
3. ~~Plain class or facet~~ **Decided 2026-09-26 (RC): plain virtual class, for
   now.** A fomo facet was the alternative, as xo-reactor2 does for its event
   sinks, but fomo has no ref-counted solution yet, and `rp<WsSender>` needs
   one. Revisit if fomo gains refcounting.

## ~~Also in scope: the session router's view of the server~~ -- superseded by issue 07

`WsSessionRouter` reaches the server through three `std::function`s:
`EndpointLookup`, `SinkFactory`, `ReplyFn`. These were first to become one
per-session interface, `WsSessionHost`. They go differently now:

- `ReplyFn` becomes the session's `WsSender` (above);
- `SinkFactory` disappears -- the router makes sinks from the sender;
- `EndpointLookup` becomes `UrlRouter const &` --
  `.xo-backlog/xo-websock/issues/07`, which pulls the endpoint maps out of
  `WebserverImpl`. The lookup was never per-session, so a per-session host
  object was the wrong shape for it.

`WsSessionHost` is not built.

## Also in scope: the stream receive function

Widened 2026-09-26 (RC). `StreamReceiveFn`
(`xo-webutil/include/xo/webutil/StreamEndpointDescr.hpp`), added in issue 04,
becomes an API class: an abstract receiver with a virtual
`receive(rp<WebsocketSink> const &, Json::Value const &)`, held by
`StreamEndpointDescr` and `DynamicEndpoint` in place of the `std::function`.
It is the one callback here that APPLICATION code implements -- the flywheel
demo's "step" handler will be one -- so it is the interface most worth making
explicit. The threading contract now documented on the alias moves to the
class.

Open: `StreamReceiveFn` has two siblings in the same descriptor,
`StreamSubscribeFn` and `StreamUnsubscribeFn`. The three together ARE a stream
endpoint's behaviour, so a single endpoint interface with subscribe,
unsubscribe and receive could replace all three -- the same one-role shape as
`WsSessionHost` above. Kept out of scope for now: it reaches
xo-reactor2websock, whose `stream_endpoint_descr()`
(`xo-reactor2websock/src/reactor2websock/reactor_endpoints.cpp`) builds the
subscribe/unsubscribe lambdas, and xo-pyreactor2websock above it.

## Not in scope, but the same objection applies

The remaining `std::function` aliases on the path:

```bash
grep -rn "std::function<" xo-websock/include xo-websock/src xo-webutil/include
```

| alias | where |
|---|---|
| `StreamSubscribeFn`, `StreamUnsubscribeFn` | `StreamEndpointDescr` (xo-webutil) -- one set per registered endpoint; see the Open note above |
| `HttpEndpointFn` | `HttpEndpointDescr` (xo-webutil) |

These are created once per registered endpoint, at startup, not per
connection. Worth deciding whether this ticket sets the pattern for them.

Also: the sink itself is still `new WebsocketSinkImpl(...)`
(`WebsocketSink.cpp:145`), and every outbound message is a `std::string` built
through a `std::stringstream`. Removing the `std::function` removes one heap
allocation per subscription, not per message. If per-message allocation is the
real target, that is a separate and larger change.

## Progress

**Step 1, 2026-09-26 -- `StreamReceiver`** (RC: smallest scope first; name and
`rp<StreamReceiver>` chosen by RC). Umbrella `b0cdf0a7`.

- New `xo-webutil/include/xo/webutil/StreamReceiver.hpp`: `class StreamReceiver
  : public ref::Refcount` with pure virtual
  `receive(rp<WebsocketSink> const &, Json::Value const &)` -- non-const, since
  a receiver (e.g. the flywheel's step handler) mutates its own state. The
  threading contract moved here from the alias; `Json::Value` and
  `WebsocketSink` stay forward-declared, so xo-webutil still has no jsoncpp
  dependency.
- `StreamReceiveFn` is gone. `StreamEndpointDescr` and `DynamicEndpoint` hold
  `rp<StreamReceiver>`; the accessor is `receiver()`. Null means "no receiver":
  `has_receive()` and the "stream does not accept messages" reply unchanged.
- No production code supplied a receive function
  (`xo-reactor2websock/src/reactor2websock/reactor_endpoints.cpp` passes three
  arguments), so nothing outside xo-webutil / xo-websock changed.
- Tests: the three receive lambdas in `xo-websock/utest/WsSessionRouter.test.cpp`
  became subclasses -- `RecordingReceiver`, `ThrowingReceiver`,
  `FrameReceiver`. Falsified with a compiling change (`DynamicEndpoint::receive`
  not calling the receiver): 4 cases fail.
- **`EndpointKind`** (RC, same step). `UrlRouter.test.cpp` had used a receiver
  purely as a marker, to tell a stream endpoint from an http one via
  `has_receive()`. Instead `DynamicEndpoint` records `EndpointKind {http,
  stream}`, set by `make_http` / `make_stream`, exposed as `kind()`; the test
  checks that and its streams carry no receiver. `http_response` asserts http;
  `subscribe` / `unsubscribe` / `receive` assert stream (and `receive` a
  non-null receiver), so a mix-up fails at the assert rather than as
  `bad_function_call`. Falsified: `make_stream` recording `http` aborts at the
  `subscribe` assert, and fails `url-router-keeps-http-and-stream-apart` at its
  kind check.

utest.websock 26 cases / 215 assertions (unchanged); umbrella 48/48;
`xo-build --sweep` ok in both stages.

Remaining: `WsSender` (the class, wiring, the retained-sink misdelivery test).

## Done when

- `WsSender` exists; `WebsocketSink::SendFn` and `WsSessionRouter::ReplyFn` are
  gone; one sender per session serves the router and all of its sinks
- `WsSessionSender` (production) and `RecordingSender` (tests) implement it
- a sink retained past its session's close cannot write into a later session
  that reuses the id -- a test reproduces the misdelivery before the fix, and
  shows it dropped after
- `WsSessionRouter` has no `std::function` members: it takes the session's
  `rp<WsSender>`, and `UrlRouter const &` from issue 07
- `StreamReceiveFn` is an API class; `StreamEndpointDescr` and
  `DynamicEndpoint` hold it; the router tests' receive handlers are subclasses
- `utest.websock` unchanged in what it covers, and green
- `xo-build --sweep` ok in both stages
