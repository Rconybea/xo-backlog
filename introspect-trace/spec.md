# introspect-trace — replay a request's work as arrows on the object graph

Status: draft 2026-10-20 -- decisions so far (RC); design not started.
Depends on `.xo-backlog/scope-sink/spec.md` (project 1), which comes first
Type: spec

Purpose (RC, 2026-10-20): practice **faithfully** animating algorithms.  Each
animation step comes from instrumentation recording what actually happened,
not from a hand-written script.  First subject: xo-websock / the introspect
example.  Next, once that is stable: the object2 / flywheel interaction.  So
nothing here should be specific to websock.

## Decisions so far (RC, 2026-10-20)

Recorded at the end of the first discussion.  Each is RC's choice unless
marked as inference.

**What the page shows**
- A request's work, drawn on the introspect graph as **arrows along the
  edges between boxes**, with **numbered badges** for the order.  Each arrow
  is labelled with the **method invoked**.
- An arrow's details on hover or click: the values that call logged -- at
  entry, its `log()` lines, at exit -- exactly as the instrumented scope
  recorded them.  The arrow itself carries only the number and the method
  (options considered: method only; everything on the arrow).

**How it is delivered**
- Request -> reply, as the introspect page works today (`{"cmd": "send",
  "msg": "refresh"}` -> a snapshot, `xo-websock/example/introspect/introspect.cpp:182-213`).
  New: the reply also carries the **trace of the work behind it**.  The page
  then has the request's full history, and replays it in the browser at
  its own pace (step, play).
- **Opt-in:** a request asks for a trace, and the page has a toggle for it.
  Untraced requests cost one check.
- Recording is scoped to one request.  It stops before the reply is
  serialised and sent, so the trace never contains its own delivery
  (inference).
- Live streaming of a continuous event feed was considered and set aside:
  the request/reply shape answers pacing, because each reply brings a
  finished recording.

**Objects the page does not draw** (provisional -- RC: "will have to get some
experience to settle")
- In the snapshot but hidden: the call is attributed to the **nearest drawn
  owner box**, and the details name the real object.  The drawing does not
  re-lay out during replay.
- Not in the snapshot (a temporary, or created and destroyed during the
  request): a transient **ghost box**, labelled with the object's type and
  overlaid near its caller (not laid out by ELK).

**Instrumentation: the producer side**
- `xo::log::scope` (`xo-ppsink/include/xo/ppsink/scope.hpp`) is the kernel.
  A scope is traced when it is **tagged with `self(this)`**.  Untagged scopes
  stay text logging only.  This matters: `xo-websock/src/websock/WebsocketSink.cpp:95`
  already has an untagged `scope` on the websocket send path, and it must
  not become a trace event.
- Opting in and out is done by **where traced scopes go**.  Leaving out
  detail is a legitimate choice.
- **Showing state over time is the instrumenter's job:** pass the values
  that show it as `scope` arguments.  Recovering arbitrary state changes
  would mean walking the whole server-side object graph at every traced
  call.  RC expects the general problem to be infeasible.

**The consumer side** -- project 1, `.xo-backlog/scope-sink/spec.md`
- `scope` emits **semantic events** (enter, log, exit; object, method,
  values) to a `ScopeSink` interface.  Today's text logging becomes one
  implementation.  Tracing is another, reached by **fan-out: ScopeSinks
  chain**.
- RC chose this over a separate `TraceSink` beside `PpSink`: it makes the
  semantic content already present in logging explicit, which is worth
  having on its own.  Folding trace events into `PpSink` was rejected for
  the reason `JsonPpSink` exists: `PpSink`'s api is text, so its meaning is
  gone (`.xo-backlog/xo-printjson/issues/11`).
- `LogState` (`xo-ppsink/include/xo/ppsink/LogState.hpp`), the per-thread
  logging state, is where the active ScopeSink(s) live.
- `scope`'s values arrive as a template pack, which cannot cross a virtual
  interface.  The trace keeps each value as its pretty-printed text (inference,
  agreed in discussion).

**Identity, and the shape of the reply**
- The printjson reply has **three parts** (RC):
  - **(a) an object map:** the objects to share, in whatever order the
    publisher finds convenient;
  - **(b) the snapshot:** an object prints inline only if it is not in (a),
    and as a ref if it is;
  - **(c) the trace:** objects as refs into (a).

  A well-formed reply has (a) and (b).  (c) can be disabled without losing
  any snapshot information.  RC's reason: deciding where an object prints
  by which path reached it first is an artificial constraint.
- **At event time the trace records identity only** (RC, 2026-10-20): the
  object's address (mapped to an id), its type, and the values the scope
  logged, as text.  Full objects are printed into (a) **at reply time, in
  their end state**.  A temporary, gone by then, gets a stub entry: id,
  type, and "not alive at end of request".
  - Why not print objects at event time: the snapshot would show mid-request
    state for any object the trace met; printing inside a traced method
    could deadlock on a guard the method already holds (printjson locks
    reflection-declared guards); mid-method state can be inconsistent.
  - The only new state during a request is text: the event records and the
    id table, in their own arena.  No lifetime entanglement.
- Address -> id is per print today (`JsonPrintState::objects_`,
  `xo-printjson/include/xo/printjson/JsonPrintState.hpp:254`).  RC's end
  state: a `PrintJsonScopeSink` that knows the `PrintJson` instance, and uses
  it to map addresses to ids.

## Open, for when this design starts

- **Address reuse within one request:** a temporary freed and another
  object allocated at the same address would share an id.  A traced scope
  does not see destruction.  Unsolved.
- **Objects printed by value inside others** (nested boxes, e.g. a session's
  `router`) have no `_id_` today.  For `self(this)` on one of them to find
  its box, the print state would also need to record by-value addresses.
  Unverified.
- Where the recorder lives: above xo-ppsink, which is arena-free by design
  (`LogState.hpp:5`) -- xo-indentlog2, xo-printjson or xo-websock.
- The protocol's opt-in flag, and the reply's field names for (a), (b), (c).
- The page side: replay controls, the first arrow (no traced caller: an entry
  arrow from outside), ghost-box placement.
