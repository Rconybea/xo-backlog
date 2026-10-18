# introspect-layout — place boxes by ownership; route member edges around them

Status: draft 2026-10-17 (licence and load failure settled by RC; awaiting approval)
Type: spec
Raised: 2026-10-17 (RC): "the biggest problem with the current layout
algorithm is that it has no hysteresis, so expanding / collapsing children
can trigger jarring layout changes"

The introspect page (`xo-websock/example/introspect`, `.xo-backlog/xo-websock/issues/10`
and `13`) lays out its whole drawing again on every redraw: one ELK `layered`
run over every box and every edge.  Opening a box changes that run's input,
and ELK may answer with a different arrangement of boxes that the change
never touched.

This spec splits the layout into two stages:

1. **Placement**: ELK, as today, but over **ownership edges only**.  Member
   edges (a ref member's arrow to the box it refers to) no longer influence
   where any box goes.
2. **Routing**: **libavoid** routes every member edge around the placed
   boxes, which it treats as fixed obstacles.

## The problem, measured

RC's repro (2026-10-17, against the `websock_ex_introspect` built in
`.build/xo-websock/example/introspect`, port 7680):

1. run the app; connect a browser: a WebserverImpl box, alone
2. press **Show all**: WebsocketSessionRecd is the left-most box,
   `/introspect` DynamicEndpoint the right-most
3. click the IntrospectReceiver box, which opens it and shows its one member,
   `websrv`
4. WebsocketSessionRecd now draws to the **right** of `/introspect`

Opening one leaf box moves boxes far away from it.

### Why: the box grows, and member edges steer the placement

Opening the receiver adds **no edge**.  A ref row's edge is drawn only when
its own ▸ is clicked.  What changes is the receiver's height (one row), and
ELK's full re-run answers that small change with a different arrangement.
Member edges take part in ELK's crossing minimisation and node placement
(`xo-websock/example/introspect/mount-origin/introspect.js:1744-1750`, all
edges handed to ELK; ownership edges get priority 10, member edges 0).  So a
small change in box size can flip their ordering.

### Spike results (2026-10-17, throwaway, not in either repo)

I measured three layouts by injecting each into the live page over the
Chrome DevTools Protocol, with RC's steps 2-3 (the receiver opened by a
dispatched click, and its opening asserted).  For each run I compared box
positions before and after the click, after removing the shift every box
shares:

| layout | boxes moving >20px beyond the shared shift | same-row pairs swapping left/right |
|---|---|---|
| today's: ELK, every edge | 10 of 14 | 5 |
| ELK interactive mode, a second pass seeded with the previous positions | 5 of 14 | 0, but an 883px vertical lurch |
| **ELK over ownership edges only** | **0 of 14** (shift 0,0) | **0** |
| **ownership-only ELK + libavoid member routing** | **0 of 19** (shift 0,0) | **0** |

The box counts differ (14 and 19) because the last run came later, with more
live sessions on the server.  The scripts (`repro.mjs`, `reproC.mjs`,
`repro2.mjs`, `reproA.mjs`) lived in a job's scratch directory and are gone.
Treat these numbers as one repro's evidence, not a general guarantee (see
**Not promised** below).

In the libavoid run, all 28 member edges route orthogonally, and none cuts
through a box.  The receiver's new `websrv` edge runs around the left side of
the drawing and up into WebserverImpl, and nothing else moves.

## Decisions (RC, 2026-10-17)

- **Option C**: layout follows ownership edges only, and member or refers
  edges do not influence placement.  RC: "may need to try alternatives over
  time.  Agree that A or C sounds best.  We can probably try C to get
  started."  Option A (keep the previous order, and seed ELK from it) stays
  available as a later strategy; this spec keeps the seam for it (below).
- **libavoid first**, ahead of a hand-written router: "let's try libavoid
  first."

## Design

### Two stages, two swappable functions

`draw_aux` (`introspect.js:1350`) keeps its shape: compute boxes and edges,
lay out, place, draw.  The layout step becomes two named functions, each
the only caller-visible seam for its stage:

- **`place_boxes(nodes, edges) -> {at, groups, size}`**: box and group
  positions in drawing coordinates, plus the drawing's size.  Today's
  implementation: ELK `layered`, with the options it has now
  (`introspect.js:1701-1726`), given **only ownership edges** (link, owns,
  nests: `is_ownership`, `introspect.js:474`).  Option A, or any later strategy,
  replaces this function and nothing else.
- **`route_edges(boxes, member_edges) -> Map(edge key -> [points])`**: one
  orthogonal polyline per member edge, in drawing coordinates.  Today's
  implementation: libavoid.  A hand-written router, or a different library,
  replaces this function and nothing else.

Neither function reads the DOM or global page state.  Each takes its inputs
as arguments and returns plain data.  That keeps each one swappable, and
lets a test call it.

### What each stage sees

- **Placement** sees every box, every group (ELK compound node, as today),
  and every ownership edge.  It does not see member edges, or the ports
  that exist only for them: today's `out_ports` and `#in` ports
  (`introspect.js:744-756`, `:1728-1739`) leave the ELK graph.
- **A drawn ownership edge** (the wanted-fallback case, `introspect.js:1689-1694`)
  keeps ELK's route.  ELK already routes it, and it sits inside the
  ownership structure ELK placed.
- **Routing** sees each box as an absolute rectangle, a group's children
  made absolute as step 3 does today (`introspect.js:1757-1767`).  It sees
  each member edge's two ends:
  - **leaves** its source box's bottom edge, at the port position
    `out_ports` computes today: row order, `out_port_gap` apart, centred.
    `out_ports` stays the only place that decides port geometry, now feeding
    the router instead of ELK;
  - **enters** its target box's top edge at `in_port_x` from the left, as
    today.
- **Group outlines are not obstacles.**  A group contains its own boxes, so
  it cannot be solid.  An edge may cross a group's background, as the spike's
  did.

### libavoid usage (learned in the spike)

- **Package**: `libavoid-js@0.4.5` (npm).  76 KB ES module
  (`dist/index.js`, `export {AvoidLib}`) plus a 485 KB `dist/libavoid.wasm`,
  loaded from jsDelivr, the same CDN as ELK (`mount-origin/index.html:8`).
  Licence: **LGPL-2.1-or-later**.  The page loads it as a separate,
  unmodified module, which is how ELK (EPL-2.0) and d3 (ISC) arrive already.
  Accepted (RC, 2026-10-17): "LGPL via CDN is fine".
- **Loading**: `await AvoidLib.load(wasm_url)` once, then
  `AvoidLib.getInstance()`.  The page loads it before the first draw.
- **Per redraw**: a `Router(OrthogonalRouting)`; one `ShapeRef` per box
  (`Rectangle(Point, Point)`); `processTransaction()`; read each
  `ConnRef`'s `displayRoute()`.  Then destroy the router, because its wasm
  memory is not garbage-collected.  **Unverified:** the spike leaked one
  router per redraw; the cleanup call (`Avoid.destroy`) needs to be found and
  checked.
- **Direction comes from pins, never from a point endpoint.** The binding
  has no `ConnEnd(point, direction)`: passing a direction aborts the wasm
  module ("program has already aborted!" on every later call).  Instead:
  - a `ShapeConnectionPin(shape, class_id, x_offset, y_offset, false, 0, dir)`
    per end, down (`2`) on the source port, up (`1`) on the target;
  - `ConnEnd(shape, class_id)` for each connector end.
- **One shared, non-exclusive entry pin per target box.** If each edge puts
  its own pin at the same spot, libavoid ignores those pins and draws a
  straight diagonal to the box centre.  In the spike that hit every box
  with two or more incoming edges.  So: one in-pin per target box,
  `setExclusive(false)`; one out-pin per source port.
- **Spacing**: `shapeBufferDistance` (clearance from boxes) and
  `idealNudgingDistance` (gap between parallel segments) are routing
  parameters.  The spike used 8 and 6.  Final values: tuned during
  implementation, against screenshots.

### Edges keep drawing as they do now

Step 4 (`introspect.js:1804-1817`) builds each edge's points from ELK's
sections today.  After this change, a member edge's points come from
`route_edges` and a drawn ownership edge's from ELK.  Everything after that
stays as it is, since both arrive as one list of absolute points:
`rounded_path`, casings, fades, the strongest-on-top sort, the parallel-edge
merge.

### Failure

- **libavoid fails to load** (offline, CDN down): the page cannot lay out
  member edges.  It shows the same kind of failure a failed ELK load causes
  today, plus an error in the status line.  No straight-line fallback: a
  silent degradation would hide that the router is missing.  Accepted (RC,
  2026-10-17): "fail visibly is fine".
- **A route libavoid cannot find**: it returns a straight segment between
  the ends.  `route_edges` passes it through, so the edge stays visible.  A
  diagonal in the drawing is the visible sign of a routing defect, which is
  how the spike found the pin problem.

## Tests

Browser tests, in `xo-websock/example/introspect/test/`, run by
`run.sh` (21 today: `sed -n '/^all_tests=(/,/)/p' run.sh`):

- **New `hysteresis.mjs`**: RC's repro as a test.  Show all, record every
  box's position, open IntrospectReceiver (asserting it opened), and record
  again.  After removing the shared shift, no box other than the receiver
  moves more than 20px, and no same-row pair swaps.  The test must fail
  against today's layout before the change; the spike says it will (10 of
  14).
- **New `routes.mjs`**: after Show all, every member edge path:
  - is orthogonal: each segment horizontal or vertical, within 0.5px;
  - starts on its source box's bottom edge and ends on its target's top edge
    at `in_port_x`;
  - cuts through no box's interior other than its own two ends (stretches
    along a box's border are allowed).

  This catches the diagonal fallback.
- **Existing 21 tests stay green.**  Three of them pin "server above its
  sessions" (`endpoint_expand.mjs:46`, `sender_expand.mjs:41`,
  `sesstable_expand.mjs:52`).  Ownership-only placement keeps that by
  construction.  The others that read edge paths (`edge_kinds.mjs` and
  more, `grep -ln 'path.edge' test/*.mjs`) check kinds and ends, not ELK's
  bend points.  Any that does read bend points gets a ruling in the plan,
  not a silent edit.

`route_edges` and `place_boxes` take plain data, so a node-side unit test
of either is possible.  This spec does not ask for one: the browser tests
cover them where they run.

## Not promised

- **Hysteresis in general.**  The spike covered one repro, where the change
  was a box growing.  Opening a box that adds **owned** children (a session,
  the session table) still changes ELK's input, and may still rearrange
  things.  Unverified.  If it does, option A goes behind `place_boxes`, a
  separate ticket.
- **Pretty routes.**  The spike's back edges (upward ones, e.g. Subscription
  → `/introspect`) take long detours under the drawing and up the side.
  Some parallel routes run close together.  Tuning (`idealNudgingDistance`,
  more space between layers now that ELK no longer reserves edge channels)
  is part of implementation.  Anything beyond tuning is a follow-up.
- **Stable routes across redraws.**  A new router per redraw re-routes from
  scratch, so routes may shift even when boxes do not.  libavoid supports
  moving shapes in one long-lived router, which might keep routes steadier.
  Not attempted here.

## Done when

- `place_boxes` and `route_edges` exist as the two seams above; ELK sees no
  member edge and no member port
- member edges route through libavoid, each router freed after its redraw
- `hysteresis.mjs` fails before the change and passes after; `routes.mjs`
  passes
- all existing introspect browser tests green (`run.sh`), and
  `xo-build --sweep` unchanged (the page is not C++, but the example ships
  with xo-websock)
