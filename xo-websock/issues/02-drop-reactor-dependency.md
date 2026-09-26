# 02 — xo-websock without xo-reactor: own sink API, adapter above both

Status: implemented 2026-09-26, awaiting review and commit in the umbrella
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

## What landed, 2026-09-26

Implemented as designed. Two small departures from the wording above:

- the sink kept its existing name, `web::WebsocketSink`, rather than
  `WebsockSink`: less churn, and no second name for one thing.
- the adapter and both endpoint builders live in namespace `xo::web`
  (`ReactorWebsocketSink`, `stream_endpoint_descr`, `http_endpoint_descr`),
  since what they produce are web endpoints.

| where | change |
|---|---|
| xo-webutil | `StreamSubscribeFn` takes `rp<web::WebsocketSink>`; forward-declared, as `reactor::AbstractSink` was |
| xo-websock | `WebsocketSink` is a `ref::Refcount` with `stream_name` / `n_in_ev` / `notify_ev_tp(TaggedPtr)` / `pretty`; server, `DynamicEndpoint`, session records use it; `reactor` dependency replaced by the `printjson` one it had been borrowing |
| xo-reactor | `AbstractSource::stream_endpoint_descr` and `AbstractEventStore::http_endpoint_descr` removed; `webutil` dependency dropped (cmake and nix) |
| xo-pyreactor | the two endpoint bindings removed |
| **xo-reactor2websock** (new) | `ReactorWebsocketSink` adapter; both builders as free functions holding their subject by `rp<>` |
| **xo-pyreactor2websock** (new) | module `xo.reactor2websock`: `stream_endpoint_descr(src, prefix)`, `http_endpoint_descr(store, prefix)` |
| registration | top `CMakeLists.txt`, `subsystem-list`, `ci.nix`, `xo.nix`, `pkgs/*.nix`; CI workflows regenerated with `xo-gen-ci` (additions only) |

**Python API change:** `src.stream_endpoint_descr(p)` and
`store.http_endpoint_descr(p)` became `xo.reactor2websock.stream_endpoint_descr(src, p)`
and `...http_endpoint_descr(store, p)`. There were no in-tree python callers:

```bash
grep -rn "stream_endpoint_descr\|http_endpoint_descr" --include=*.py . | grep -v '\.build/'
```

### Two install-only defects this surfaced

Both were pre-existing and invisible in the umbrella build, which shares one
cmake context and a build-tree runpath. xo-reactor2websock was the first
standalone C++ consumer of an installed xo-websock, and its python smoke test
the first thing to load one.

1. **`websockConfig.cmake` never declared jsoncpp.** `jsoncpp_lib` was in
   websock's `INTERFACE_LINK_LIBRARIES`, so a consumer that could not resolve
   the target linked a bare `-ljsoncpp_lib` and failed. Fixed:
   `find_dependency(jsoncpp CONFIG)` in `xo-websock/cmake/websockConfig.cmake.in`.
2. **Loading an installed xo-websock failed on `libssl.so.3`.** This was true
   of plain `import xo.websock` through `~/local/bin/xo-python` before any of
   this work. The root cause is in libwebsockets' exported cmake target, which
   lists openssl by full path in its PUBLIC link interface although openssl is
   its private dependency. Everything linking it therefore recorded direct
   `NEEDED libssl.so.3` / `libcrypto.so.3` entries while calling no ssl
   symbol, and the installed runpath could not resolve them. libwebsockets'
   OWN runpath does include openssl; it is only the spurious direct entries
   that break. Fixed in `xo-websock/src/websock/CMakeLists.txt`, two ways:
   - libwebsockets is linked PRIVATE, with its include directories re-exported
     PUBLIC (`$<TARGET_PROPERTY:websockets_shared,INTERFACE_INCLUDE_DIRECTORIES>`,
     since `Webserver.hpp` includes `<libwebsockets.h>`). Consumers compile
     against it but never link it, so the list stops at xo-websock.
   - `LINKER:--as-needed` PRIVATE on websock itself, guarded `NOT APPLE`
     (ld64 does not take it, and install-name linkage makes the problem moot
     there), so libwebsock.so does not record the entries either.

```bash
for f in ~/local/lib/libwebsock.so ~/local/lib/libreactor2websock.so; do
    readelf -d $f | grep -c 'NEEDED.*libssl'; done        # 0 and 0
~/local/bin/xo-python -c 'import xo.websock, xo.reactor2websock'
```

`xo-pyreactor2websock` also needed `xo_emit_python_wrapper` for standalone
builds, as xo-pyobject2 has, because it is the first python subsystem modelled on
xo-pywebsock, which has no python tests.

### Tests

`utest.reactor2websock`, 5 cases: adapter forwarding; subscribe/unsubscribe
through a stream endpoint; `/snap` suffix; and that each endpoint keeps its
source/store alive, which pins the fix for the raw-`this` capture.

Each was falsified with a change that compiles: forwarding removed; unsubscribe
made a no-op; `/snap` dropped; raw-pointer capture of the source; raw-pointer
capture of the store. Each failed at its intended assertion. A first attempt at
the raw-source variant did NOT compile. The harness flagged it rather than
reporting the stale binary's result -- the trap from
`.xo-backlog/xo-facet/issues/02`.

`utest.pyreactor2websock`, 4 cases: module import (which pulls xo.reactor and
xo.webutil), the builders gone from xo.reactor, and argument type checks.
Nothing at the xo.reactor level is constructible from python, so no event
flows here. Not falsified.

### Verification

```bash
xo-deps --why=xo-websock:xo-reactor -q; echo $?   # 1
xo-deps --why=xo-reactor:xo-webutil -q; echo $?   # 1
comm -23 <(xo-deps --deps-of=xo-websock --format=names -q | sort) \
         <(xo-deps --deps-of=xo-printjson --format=names -q | sort)
# xo-callback xo-websock xo-webutil
```

Umbrella ctest 47/47. `xo-build --sweep` ok in both stages, after reinstalling
xo-cmake so the installed subsystem-list carried the two new entries:

```
stage 1: 73 attempted: 73 ok, 0 with no tests, 0 failed, 0 skipped
stage 2: 73 attempted: 46 ok, 27 with no tests, 0 failed, 0 skipped
```

The +2 in each stage is the two new subsystems.

**Not verified:** `nix-build ci.nix -A xo-pyreactor2websock` (or any of the four
changed packages). The nix files follow existing patterns, and the python
test's PYTHONPATH chain follows xo-pyobject2's, but nothing has built them.

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
