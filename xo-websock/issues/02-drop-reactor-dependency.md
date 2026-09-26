# 02 — xo-websock without xo-reactor: own sink API, adapter above both

Status: open
Type: refactor / levelization
Raised: 2026-09-26, while planning AllocFlywheel visualization

## Why

The goal behind the AllocFlywheel visualization is an animation that **provably
matches the code, because it inspects the code directly** -- unlike the
xo-arena documentation animations, which carry their own model of the data
structure and can drift from it. That means a documentation page for subsystem
X embeds a live view served by a demo app that depends on X and on xo-websock.

So xo-websock's level is part of the documentation's leveling. Today a live
demo for xo-arena or xo-facet would drag in the whole reactor stack, because
every one of these arrives through a single edge:

```bash
for s in xo-ordinaltree xo-object xo-alloc xo-allocutil; do
    xo-deps --why=xo-websock:$s -q; done
# xo-websock -> xo-reactor -> xo-ordinaltree [-> xo-object | xo-alloc | xo-allocutil]

comm -23 <(xo-deps --deps-of=xo-websock   --format=names -q | sort) \
         <(xo-deps --deps-of=xo-printjson --format=names -q | sort)
# 2026-09-26: xo-alloc xo-allocutil xo-callback xo-object xo-ordinaltree
#             xo-ratio xo-reactor xo-unit xo-websock xo-webutil
```

After this ticket, the second command should list only `xo-callback`,
`xo-websock` and `xo-webutil`. (`xo-webutil` is already clean -- it only
forward-declares `reactor::AbstractSink`.)

## The coupling is one interface wide

xo-websock uses reactor for exactly one thing: `WebsocketSinkImpl` implements
`reactor::AbstractSink` (`xo-websock/src/websock/WebsocketSink.cpp`), so a
reactor source can attach to it. What the sink actually does is reactor-free:
`notify_ev_tp(TaggedPtr)` -> `PrintJson` -> write to the session. The server
holds it as `rp<AbstractSink>` (`xo-websock/src/websock/Webserver.cpp:330`,
`:359`, created at `:1405`).

xo-websock does not need reactor's typed-sink formalism: it already assumes
dynamic dispatch from `TaggedPtr`. The formalism is good and stays in
xo-reactor; xo-websock just stops depending on it.

```bash
grep -rn reactor xo-websock/include xo-websock/src --include=*.hpp --include=*.cpp
```

## The less obvious half: xo-reactor builds web endpoints itself

`xo-reactor -> xo-webutil` is a real edge, used in two places:

| site | builds |
|---|---|
| `AbstractSource::stream_endpoint_descr` -- `xo-reactor/src/reactor/AbstractSource.cpp:21` | `StreamEndpointDescr` whose subscribe fn takes `rp<reactor::AbstractSink>` |
| `AbstractEventStore::http_endpoint_descr` -- `xo-reactor/include/xo/reactor/EventStore.hpp:58` | `HttpEndpointDescr` for `<prefix>/snap` |

```bash
xo-deps --why=xo-reactor:xo-webutil -q
grep -rn "webutil\|EndpointDescr" xo-reactor/include xo-reactor/src
```

The stream one CANNOT stay: once `StreamSubscribeFn` takes a websock sink,
reactor cannot build it without seeing xo-websock. The http one could stay and
keep compiling; **decided 2026-09-26 to move both**, so xo-reactor drops
xo-webutil entirely and no longer knows the web exists.

## Design (decided 2026-09-26)

1. **xo-websock gains its own sink API** taking a `TaggedPtr` -- working name
   `WebsockSink`. The server, and `StreamSubscribeFn` in xo-webutil, speak it.
   No reactor types anywhere in xo-websock or xo-webutil.
2. **New subsystem `xo-reactor2websock`**, above `{xo-reactor, xo-websock}`.
   Read the name as reactor-*to*-websock, not as xo-reactor2. It holds:
   - the `WebsockSink` -> `reactor::AbstractSink` adapter: the reactor half of
     today's `WebsocketSinkImpl`, lifted out of xo-websock
   - `stream_endpoint_descr(rp<AbstractSource>, url_prefix)` as a free
     function, lifted out of `AbstractSource`
   - `http_endpoint_descr(rp<AbstractEventStore>, rp<PrintJson>, url_prefix)`
     as a free function, lifted out of `AbstractEventStore`.
     `http_snapshot` stays on the store -- it needs only PrintJson and an
     ostream.
3. **New subsystem `xo-pyreactor2websock`** wraps it. xo-pyreactor's bindings of
   both endpoint builders (`xo-pyreactor/src/pyreactor/pyreactor.cpp:60`,
   `:95`) move there.

The analogous `{xo-reactor2, xo-websock}` adapter is **deferred**. When it
happens it goes into the same library. xo-reactor2 already has the facet it
would adapt to -- `AEventSink` (`xo-reactor2/include/xo/reactor2/detail/AEventSink.hpp`)
has the same shape as `reactor::AbstractSink`, down to
`notify_ev_tp(TaggedPtr)` -- and xo-reactor2 does not reach xo-ordinaltree:

```bash
xo-deps --why=xo-reactor2:xo-ordinaltree -q; echo $?   # 1 = no path
```

## Principle to hold the design to

**xo-websock carries frames and never shapes them.** Everything the animation's
correctness depends on lives in the frame producer -- PrintJson and the bespoke
printers in xo-printjson -- which is already transport-free. The eventual
in-browser (wasm) mode will replace the transport wholesale, since libwebsockets
will not exist there. If the producer stays out of xo-websock, live-server mode
and in-browser mode run the identical producer and only the pipe differs.

## Incidental findings, deliberately not acted on here

- **Existing race and lifetime hazard in `http_endpoint_descr`.** Its body
  already says *"WARNING: race condition here, given webserver runs from a
  separate thread"* (`EventStore.hpp:71`), and the lambda captures a raw
  `this`, so the endpoint can outlive the store. Moving it to a free function
  taking `rp<AbstractEventStore>` fixes the lifetime half. The race is the same
  one the AllocFlywheel visualization has to answer, and is not solved by this
  ticket.
- **jsoncpp does one job:** parsing the inbound `{"cmd": "subscribe",
  "stream": ...}` message in `WebserverImpl::perform_ws_cmd`
  (`Webserver.cpp:1359`). Direction is to retire it eventually; deferred.
- **Doc comment vs code on the key name.** The comments at `Webserver.cpp:796` and
  `:1363` say `"command"`; the code reads `root["cmd"]`. Probably `cmd` is right. Also,
  a parse failure is logged and then processing continues on an empty `root`.
  Harmless (nothing matches), but it makes a page that gets no frames silent.
- The kalman demo in `xo-websock/utest` calls both endpoint builders
  (`websock_utest_main.cpp:245-252`) but is not built -- see issue 01. Porting
  it, if chosen, would target the new free functions.

## Blast radius

```bash
xo-deps --users-of=xo-websock --format=names -q    # xo-pywebsock xo-websock
xo-deps --users-of=xo-webutil --format=names -q
grep -rn "stream_endpoint_descr\|http_endpoint_descr" --include=*.hpp --include=*.cpp \
     --include=*.py . | grep -v '\.build/'
```

## Done when

- `xo-deps --why=xo-websock:xo-reactor -q` exits 1
- `xo-deps --why=xo-reactor:xo-webutil -q` exits 1
- the `comm` above lists only `xo-callback`, `xo-websock`, `xo-webutil`
- `xo-reactor2websock` and `xo-pyreactor2websock` exist, are in
  `xo-cmake/etc/xo/subsystem-list`, and are actually swept (the list is read
  from the INSTALLED copy -- see CONVENTIONS, the `comm` against
  `~/local/share/etc/xo/subsystem-list`)
- the endpoint builders are reachable from python through
  `xo-pyreactor2websock`
- `xo-build --sweep` ok in both stages
