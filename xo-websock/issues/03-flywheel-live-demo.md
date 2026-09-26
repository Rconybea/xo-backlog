# 03 — live browser view of an AllocFlywheel, driven by the real code

Status: open -- design mostly OPEN, deliberately; see "Open"
Type: feature
Raised: 2026-09-26
Blocked by: `.xo-backlog/xo-websock/issues/02` (xo-websock without xo-reactor)

This ticket preserves what planning conversations established so the thread can
be picked back up after issue 02. It does NOT settle the demo's design; the
open questions are listed at the end, and they are genuinely open.

## The goal, and why it is harder than it looks

Animate an AllocFlywheel's state in a browser, **from the running code itself**.

The xo-arena documentation already has animations, but each carries its own
model of how the data structure works, so the animation and the code can drift
apart unnoticed. The property wanted here is an animation that provably matches
xo-foo because it inspects xo-foo directly.

The price is that a documentation site showing such an animation cannot be
static. It either talks to an xo app running somewhere, or runs xo in a
browser sandbox (wasm). The sandbox is the eventual direction, and is expected
to get entangled with platform idiosyncrasies, so it will take longer than it
looks. The live-server route comes first.

Leveling applies to documentation as much as code: a live demo for subsystem X
depends on X and on xo-websock. That is why issue 02 comes first: it takes
xo-websock down to xo-printjson's level.

## Principle: the transport carries frames and never shapes them

Everything the animation's correctness depends on lives in the **frame
producer**, i.e. PrintJson plus the bespoke printers in
`xo-printjson/src/printjson/PrintJson.cpp` (`JsonPrinter_ObjectSlot` :547,
`JsonPrinter_RootSet` :648, `JsonPrinter_AllocFlywheel` :733). The producer is
already transport-free. The wasm mode will replace the transport wholesale,
since libwebsockets does not exist there. Keeping the producer out of
xo-websock means live-server and in-browser modes run the identical producer.

## What already exists (data side is complete)

`o.flywheel_frame(fw)` (python, `xo-pyobject2/src/pyobject2/pyobject2.cpp:135`),
or `PrintJson().print(fw, &os)` in C++, yields:

```json
{"_name_": "Flywheel",
 "pools": [ {"_name_": "MemorySizeInfo", "name": ..., "used": ..., "allocated": ...,
             "committed": ..., "reserved": ..., "lo": ..., "hi": ...}, ... ],
 "strong": {"_name_": "RootSet", "size": 2, "capacity": 511, "live": 2, "free": [],
            "slots": [{"_name_": "ObjectSlot", "typeseq": 10, "type": "xo::scm::DFloat",
                       "offset": 16, "size": 8}, ...]}}
```

Semantics that are easy to get wrong from outside:

- **A frame is read at PRINT time, not snapshot time.** `snapshot()` was
  retired along with the Info classes (`.xo-backlog/xo-facet/issues/02`). A
  frame is only meaningful while the flywheel is alive and unmutated during
  serialization.
- **Slot `offset` is relative to the slot's OWN arena** (via
  `DArena::obj2arena`), not to `pools[0]`.
- **Slot `size` is the allocation size** from the alloc header
  (`AllocInfo::size()`; includes padding, excludes the header). It is not
  `sizeof(DRepr)`, which would be wrong for `DArray` and `DString`. Why both
  `obj2arena` and `alloc_info` are sound: `.xo-backlog/xo-facet/issues/04`.
- An empty slot renders as `null` and is still emitted, so a slot's index is
  its array position. `free` is in push order; reuse pops the back.
- `reserved` and `capacity` differ by host page size (4k linux, 16k darwin).
- **Wire contract** is pinned in `xo-object2/utest/flywheel_frame.test.cpp`.
  Change a key only deliberately, updating that test.

Transport pieces in xo-websock: `register_http_endpoint` (snapshot per GET) and
`register_stream_endpoint` (push over websocket). A kalman-era d3-over-websocket
page survives in `xo-websock/utest/mount-origin/` (`ex_websock.html`,
`ex_websock.js`). It is not built (issue 01), but it is worth mining for the
browser side.

## The hazard any design must answer

`Webserver` runs `lws_service` on its own `std::thread`
(`xo-websock/src/websock/Webserver.cpp:1231`). If anything mutates the
flywheel on another thread while a frame is being serialized, the
`DArenaVector<ObjectSlot>` can reallocate under the walk: a use-after-free,
not a wrong number. It is the same shape as the `typerecd` vector argument in
`.xo-backlog/xo-facet/issues/03`.

This is not new. `AbstractEventStore::http_endpoint_descr` has carried
*"WARNING: race condition here, given webserver runs from a separate thread"*
since 2022 (see issue 02).

## Driver options considered (2026-09-25/26)

1. **Self-contained C++ example.** It builds a flywheel and allocates and
   releases on a timer, with serialization on the same thread as mutation, so
   there is no race by construction.
2. **Python drives, server on its own thread.** This is the end goal
   (`o.Float(1.5)` in a REPL and the animation reacts), but it needs the race
   answered first: a lock that both sides take, or the mutating side producing
   the frame string and handing over only that.
3. **http snapshot only, browser polls.** The smallest change, but polling is
   not really animation, and the race narrows rather than goes away.

**Decided in principle: "a variation on 1".** What the variation is was not
settled.

A suggestion from the discussion, not decided: if the example has the
*mutating* side produce the frame string and hand it to the sink, moving
mutation into python later changes who calls "produce frame", and not the
transport or the page.

## Consequence of issue 02 for the timer

Option 1 was going to take its tick from xo-reactor. After issue 02,
xo-websock has no reactor, so the tick comes either from libwebsockets' own
scheduled callbacks (`lws_sul`) or from a main-loop tick. `lws_sul` would run
the tick on the service thread, which would also sidestep the race. That is
unverified, and xo-websock does not use `lws_sul` today:

```bash
grep -rn lws_sul xo-websock/src    # empty as of 2026-09-26
```

## Related, not required

- **Frame sequence number:** not done. `JsonPrinter_AllocFlywheel` is where it
  would go. An animation wants one to notice dropped or reordered frames
  instead of interpolating across a gap.
- **`.xo-backlog/xo-printjson/issues/04`, JSON string escapes:** latent while
  type names are the only strings in a frame.
- `import xo.object2` prints `SetupObject2::register_facets`'s debug log, a
  side effect of re-enabling that call for `xo-facet/01`. This would be noise
  in a REPL-driven demo.

## Open

- What "a variation on 1" means concretely.
- Where the demo lives: under `xo-websock/example/`, or a demo subsystem above
  xo-facet + xo-websock (which the documentation-leveling argument may favour).
- What gets animated first (root set? pools? slot lifecycle?) and at what frame
  rate. The empty-flywheel frame is ~500 bytes, so rate matters.
- The browser side: port pieces of the kalman-era d3 page, or start fresh.
- How the python-driven step (option 2) will answer the race.

## Done when

To be written once the Open items are settled. Until then this ticket is a
record, not a plan.
