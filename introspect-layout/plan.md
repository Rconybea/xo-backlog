# introspect-layout Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** stop the introspect page from rearranging distant boxes when a box opens or closes.  ELK places boxes by ownership edges only, and libavoid routes member edges around the placed boxes.

**Architecture:** the page's single layout step splits into two seams.  `place_boxes` (ELK `layered`, ownership edges only) and `route_edges` (libavoid, orthogonal, around fixed boxes) each take plain data and return plain data.  Drawing (step 4 onward in `draw_aux`) keeps consuming one list of absolute points per edge.

**Tech Stack:** browser JavaScript (classic script, no bundler): d3 7.9.0, elkjs 0.12.0, libavoid-js 0.4.5 (ES module + wasm, jsDelivr).  Browser tests: node 22 + headless chrome over CDP (`xo-websock/example/introspect/test/run.sh`).

**Spec:** `.xo-backlog/introspect-layout/spec.md` (approved by RC 2026-10-17).  Read it first; this plan argues from it.  Its "libavoid usage" section records what the spike learned; it binds the implementation.

**How this plan is written:** behaviour, interfaces, invariants and named risks.  Tests are given as code, because they ARE the specification.  Implementation is NOT given as code: a plan carrying function bodies blinds per-task review, since reviewer and implementer would read the same source.

## Global Constraints

- **Never `git commit` / `git push` in the umbrella repo** (`~/proj/xo-umbrella2`).  Each task ends by stopping for RC to review and commit.  Backlog files (`~/proj/xo-backlog`, the `.xo-backlog` symlink) may be committed and pushed.
- **The page is served from a build-time COPY** of `mount-origin/`.  Before every browser-test run, refresh the copy and the server:
  `cmake --build .build --target websock_ex_introspect -j16` (from `~/proj/xo-umbrella2`).
- **Run browser tests:** `xo-websock/example/introspect/test/run.sh .build/xo-websock/example/introspect/websock_ex_introspect [TEST..]` (no TEST = all).  Each test gets a fresh server.  Logs and screenshots go to the temp directory the last line names.  Never redirect its output to `/dev/null`.
- **libavoid:** `libavoid-js@0.4.5`.  Module `https://cdn.jsdelivr.net/npm/libavoid-js@0.4.5/dist/index.js` (exports `{AvoidLib}`), wasm `https://cdn.jsdelivr.net/npm/libavoid-js@0.4.5/dist/libavoid.wasm`.  Load once: `await AvoidLib.load(wasm_url)`, then `AvoidLib.getInstance()`.  LGPL-2.1-or-later via CDN: accepted by RC.
- **libavoid traps (spec, "libavoid usage"):**
  - never pass a direction to `ConnEnd(point, dir)`: it aborts the wasm module for good.  Set directions on `ShapeConnectionPin(shape, class_id, x_off, y_off, false, 0, dir)` (down = 2, up = 1), and connect with `ConnEnd(shape, class_id)`;
  - one shared, non-exclusive in-pin per target box (`setExclusive(false)`), one out-pin per source port.  Per-edge pins at one spot produce diagonals to the box centre.
- **Load failure is visible, with no fallback** (RC, 2026-10-17): the status line (`#status`) shows an error naming libavoid, and the page draws nothing.
- **No ambient state for the seams:** `place_boxes` and `route_edges` take every input as an argument (including the libavoid instance) and read no DOM or page globals.
- **Style:** match `introspect.js`: `snake_case` names, `/** ... **/` doc comments on functions, comment density as in its neighbours, cite `.xo-backlog/introspect-layout/spec.md` where a reader needs the reason.

### Test preamble

Every new browser test starts with this preamble, the one the existing tests use (`test/receiver.mjs:3-15`), unchanged:

```js
import fs from "node:fs";
const [,, cdp_port, port, out] = process.argv;
const sleep = (ms) => new Promise(r => setTimeout(r, ms));
const tgt = await (await fetch(`http://localhost:${cdp_port}/json/new?http://localhost:${port}/`, {method: "PUT"})).json();
const ws = new WebSocket(tgt.webSocketDebuggerUrl); await new Promise(r => ws.onopen = r);
let seq = 0; const pending = new Map();
ws.onmessage = (ev) => { const m = JSON.parse(ev.data); if (m.id && pending.has(m.id)) { pending.get(m.id)(m); pending.delete(m.id); } };
const call = (method, params = {}) => new Promise(r => { const id = ++seq; pending.set(id, r); ws.send(JSON.stringify({id, method, params})); });
const ev = async (expr) => { const r = await call("Runtime.evaluate", {expression: expr, returnByValue: true, awaitPromise: true}); if (r.result.exceptionDetails) console.log("EXC", JSON.stringify(r.result.exceptionDetails).slice(0, 300)); return r.result.result?.value; };
let ok = true; const check = (c, msg) => { console.log(`${c ? "ok  " : "FAIL"} ${msg}`); if (!c) ok = false; };
await call("Emulation.setDeviceMetricsOverride", {width: 1500, height: 1000, deviceScaleFactor: 1, mobile: false});
for (let i = 0; i < 100 && !(await ev(`typeof last_event !== "undefined" && !!last_event && document.querySelectorAll("g.node").length >= 1`)); i++) await sleep(100);
await sleep(500);
```

and ends with:

```js
fs.writeFileSync(out, Buffer.from((await call("Page.captureScreenshot", {format: "png"})).result.data, "base64"));
console.log(ok ? "ALL OK" : "SOME FAILED");
ws.close(); process.exit(ok ? 0 : 1);
```

## Review Focus

These are the inputs the spec implies but doesn't name that are most likely to bite, most likely first.  Each has its test in the owning task, marked **[RF n]**.

1. **The first snapshot arrives before libavoid has loaded.**  The websocket subscription can answer in milliseconds, and the wasm is 485 KB.  Expect: the first draw waits for the load and then draws normally, with no error and no edges missing.  Task 3 (`avoid_load.mjs`, delayed mode).
2. **A member edge with an end inside a group** (e.g. from a session's router, drawn nested in the session).  ELK reported such routes relative to the group; `route_edges` takes absolute boxes.  Expect: the route starts and ends on the drawn boxes, not offset by the group's position.  Task 3 (`routes.mjs` covers it after Show all; Task 1 checks that such an edge exists in the run).
3. **Two or more edges into one box**, the case that produced diagonals in the spike.  Expect: every route is orthogonal and ends at the box's one entry point.  Task 2 (`route_edges.mjs`, case b) and Task 3 (`routes.mjs`).
4. **A member edge whose source and target are the same box** (an object referring to itself, or to an object drawn nested in its own box).  Unverified whether the live graph can produce one.  Expect: `route_edges` returns an orthogonal route, does not throw, and does not abort the wasm.  Task 2 (`route_edges.mjs`, case c).
5. **A shown box with no ownership edge to any other shown box.**  Without member edges, ELK would see it as a separate connected component and pack it on its own.  Unverified whether the visibility rules (`introspect.js:338-380`) allow it.  Expect: placement stays deterministic, and the hysteresis check holds.  Task 3, Step 1 (look for the case, and ledger what was found).

---

### Task 1: The failing hysteresis test, and the route guard

**Files:**
- Create: `xo-websock/example/introspect/test/hysteresis.mjs`
- Create: `xo-websock/example/introspect/test/routes.mjs`
- Modify: `xo-websock/example/introspect/test/run.sh` (append `hysteresis routes` to `all_tests`)

**Interfaces:**
- Consumes: page globals the existing tests already use: `last_event`, `settled()`, `expanded`, `drawn_at` (Map id → `{x, y, w, h}`, drawing coordinates incl. `pad`), `in_port_x`, `pad`; `path.edge.member` elements, whose `__data__` carries `pts` (drawing coordinates WITHOUT `pad`), `from` (source box id) and `target` (target box id).
- Produces: two tests that Task 3 must turn or keep green.  `hysteresis` FAILS today; `routes` PASSES today and must still pass after the router swap.

- [ ] **Step 1: Write `hysteresis.mjs`** (RC's repro, spec "The problem, measured")

```js
// RC's repro (.xo-backlog/introspect-layout/spec.md): Show all, then open
// IntrospectReceiver.  Opening one leaf box must not move the others.
// <Test preamble>
const R = "stream:/introspect:receiver";
const G = (id) => `[...document.querySelectorAll("g.node")].find(g => g.__data__.id === ${JSON.stringify(id)})`;
await ev(`document.getElementById("show-all").click(); 1`); await ev(`settled()`);
const before = new Map(await ev(`[...drawn_at]`));
check(before.size >= 10 && before.has(R), `Show all drew the graph, receiver included: ${before.size} boxes`);
// the user's click, on the box itself (not on screen coordinates: the legend may cover it)
await ev(`${G(R)}.dispatchEvent(new MouseEvent("click", {bubbles: true})); 1`); await ev(`settled()`);
check(await ev(`expanded.has(${JSON.stringify(R)})`), "the click opened the receiver");
const after = new Map(await ev(`[...drawn_at]`));
const ids = [...before.keys()].filter(k => k !== R && after.has(k));
const med = (xs) => { const s = [...xs].sort((a, b) => a - b); return s[Math.floor(s.length / 2)]; };
const sx = med(ids.map(k => after.get(k).x - before.get(k).x)), sy = med(ids.map(k => after.get(k).y - before.get(k).y));
const moved = ids.filter(k => Math.abs(after.get(k).x - before.get(k).x - sx) > 20 || Math.abs(after.get(k).y - before.get(k).y - sy) > 20);
const swaps = [];
for (const a of ids) for (const b of ids)
  if (a < b && Math.abs(before.get(a).y - before.get(b).y) < 5 && Math.abs(after.get(a).y - after.get(b).y) < 5
      && Math.sign(before.get(a).x - before.get(b).x) !== Math.sign(after.get(a).x - after.get(b).x))
    swaps.push([a, b]);
console.log(`   shared shift (${sx},${sy}); moved: ${JSON.stringify(moved)}; swaps: ${JSON.stringify(swaps)}`);
check(moved.length === 0, `no other box moves more than 20px beyond the shared shift: ${moved.length} of ${ids.length} did`);
check(swaps.length === 0, `no same-row pair swaps left/right: ${swaps.length} did`);
// <Test ending>
```

- [ ] **Step 2: Build, then run it against today's layout**

Run: `cmake --build .build --target websock_ex_introspect -j16 && xo-websock/example/introspect/test/run.sh .build/xo-websock/example/introspect/websock_ex_introspect hysteresis`
Expected: `hysteresis: FAIL`, with `moved` non-empty.

**Known risk:** the spike reproduced this on RC's server with several live sessions.  A fresh server with one browser connection draws a different graph, and an earlier probe in that state did NOT reproduce it.  If the test PASSES here, it proves nothing yet.  Add live sessions before Show all, the way RC's server had them: open further CDP targets on the same page URL (`/json/new?http://localhost:${port}/`, one per extra session; two extra matched the spike).  Wait until `last_event`'s session table lists them, and re-run.  Keep the smallest number of extra sessions that fails, and ledger it as a Ruling.  If no session count fails, stop and report: the test cannot be made RED, and that is RC's call.

- [ ] **Step 3: Write `routes.mjs`** (the router contract; spec "Tests")

```js
// every member edge: orthogonal, from its source box's bottom edge to its
// target's top edge at in_port_x, through no other box
// (.xo-backlog/introspect-layout/spec.md)
// <Test preamble>
await ev(`document.getElementById("show-all").click(); 1`); await ev(`settled()`);
const box = new Map(await ev(`[...drawn_at]`));
const P = 20;   // draw_aux's pad, local there (introspect.js:1380): drawn_at includes it, pts do not
const IN = await ev(`in_port_x`);
const edges = await ev(`[...document.querySelectorAll("path.edge.member")].map(p => ({from: p.__data__.from, to: p.__data__.target, pts: p.__data__.pts.map(q => ({x: q.x + ${P}, y: q.y + ${P}}))}))`);
// problems with one route; [] if it keeps the contract
const problems = (e) => {
  const out = [], s = box.get(e.from), t = box.get(e.to), pts = e.pts, a = pts[0], z = pts[pts.length - 1];
  if (!s || !t) return [`no box for ${e.from} -> ${e.to}`];
  for (let i = 1; i < pts.length; i++)
    if (Math.abs(pts[i].x - pts[i - 1].x) > 0.5 && Math.abs(pts[i].y - pts[i - 1].y) > 0.5) out.push(`diagonal segment ${i}`);
  if (Math.abs(a.y - (s.y + s.h)) > 1.5 || a.x < s.x || a.x > s.x + s.w) out.push(`starts off its source's bottom edge: ${JSON.stringify(a)}`);
  if (Math.abs(z.y - t.y) > 1.5 || Math.abs(z.x - (t.x + IN)) > 1.5) out.push(`ends off its target's entry point: ${JSON.stringify(z)}`);
  for (const [id, b] of box) {
    if (id === e.from || id === e.to) continue;
    for (let i = 1; i < pts.length; i++) {   // a segment strictly inside the box, 1px in from its border
      const x0 = Math.min(pts[i - 1].x, pts[i].x), x1 = Math.max(pts[i - 1].x, pts[i].x);
      const y0 = Math.min(pts[i - 1].y, pts[i].y), y1 = Math.max(pts[i - 1].y, pts[i].y);
      if (x1 > b.x + 1 && x0 < b.x + b.w - 1 && y1 > b.y + 1 && y0 < b.y + b.h - 1) out.push(`crosses ${id}`);
    }
  }
  return out;
};
// the checker catches what it exists for: a diagonal through a box
const ids = [...box.keys()];
check(problems({from: ids[0], to: ids[1], pts: [{x: box.get(ids[0]).x, y: box.get(ids[0]).y}, {x: box.get(ids[1]).x + 7, y: box.get(ids[1]).y + 9}]}).length > 0, "self-check: a diagonal is reported");
check(edges.length >= 10, `Show all drew member edges: ${edges.length}`);
// [RF 2] at least one edge with an end inside a group (ELK reported those relative to the group) is in the run
check(await ev(`[...document.querySelectorAll("path.edge.member")].some(p => { const g = (id) => [...document.querySelectorAll("g.node")].find(n => n.__data__.id === id)?.__data__.group;
  return g(p.__data__.from) !== undefined || g(p.__data__.target) !== undefined; })`), "[RF 2] an edge with an end inside a group is covered");
const bad = edges.map(e => [e.from, e.to, problems(e)]).filter(x => x[2].length);
for (const b of bad) console.log("   ", JSON.stringify(b));
check(bad.length === 0, `every member edge keeps the route contract: ${bad.length} of ${edges.length} do not`);
// <Test ending>
```

A box drawn inside a group carries `group` (the id of the group's top box; `introspect.js:431-443`).  `P = 20` copies `draw_aux`'s local `pad`.  If `pad` changes, this test's endpoint checks fail loudly, which is the right outcome.

- [ ] **Step 4: Run it against today's layout**

Run: the Step 2 command with `routes`.
Expected: `routes: ok`.  This test guards the swap: ELK's routes keep the contract today, and libavoid's must too.  It is not a RED test; its self-check line is what shows it can fail.  If it FAILS today, read which contract today's routes break.  Ledger a Ruling: either the contract is wrong (e.g. ELK's in-port sits at `y = -1`, outside 1.5px), or today's routes really break it.  Never loosen a tolerance without a ruling.

- [ ] **Step 5: Append both to `all_tests` in `run.sh`, and run the whole suite**

Run: the Step 2 command with no TEST.
Expected: `22 passed, 1 failed` (21 existing + `routes`; `hysteresis` fails).

- [ ] **Step 6: Stop for RC to review and commit.**

---

### Task 2: `route_edges`: libavoid behind one function

**Files:**
- Create: `xo-websock/example/introspect/mount-origin/route_edges.js`
- Modify: `xo-websock/example/introspect/mount-origin/index.html` (load `route_edges.js` before `introspect.js`, line 131)
- Create: `xo-websock/example/introspect/test/route_edges.mjs`
- Modify: `xo-websock/example/introspect/test/run.sh` (append `route_edges`)

**Interfaces:**
- Produces (classic-script globals, so `introspect.js` and the tests can reach them):
  - `load_router() -> Promise<Avoid>`: imports the libavoid module (`import()` works from a classic script), loads the wasm, and resolves to `AvoidLib.getInstance()`.  It rejects with an `Error` whose message names libavoid and the failing URL.  It does not touch the DOM; Task 3 decides what a rejection shows.
  - `route_edges(avoid, boxes, edges) -> Map<string, Array<{x, y}>>`:
    - `avoid`: the instance `load_router()` resolved to;
    - `boxes`: `Map<id, {x, y, w, h}>`, absolute, one coordinate frame;
    - `edges`: `Array<{key, from, out_x, to}>`.  `from` and `to` are box ids, and `out_x` is the out-port's x offset from the source box's left edge;
    - returns, per `key`, an orthogonal polyline in the same frame as `boxes`.  The first point is `(from.x + out_x, from.y + from.h)`; the last is `(to.x + in_x, to.y)`, where `in_x` is a fourth parameter, `route_edges(avoid, boxes, edges, in_x)`.  Task 3 passes `in_port_x`.
- Invariants:
  - the router `route_edges` creates is freed before it returns, on every path, including a throw (`try`/`finally`);
  - `route_edges` is synchronous: no `await` inside, so a superseded redraw cannot interrupt it half-way;
  - spacing parameters: `shapeBufferDistance` 8 and `idealNudgingDistance` 6 (the spike's values; Task 4 tunes them), named constants at the top of the file.

- [ ] **Step 1: Write `route_edges.mjs`**: synthetic boxes, no server graph needed

```js
// route_edges (.xo-backlog/introspect-layout/spec.md): libavoid behind one
// function, on synthetic boxes
// <Test preamble>
const r = await ev(`(async () => {
  const A = await load_router();
  const orth = (pts) => pts.every((p, i) => i === 0 || Math.abs(p.x - pts[i - 1].x) < 0.5 || Math.abs(p.y - pts[i - 1].y) < 0.5);
  const crosses = (pts, b) => pts.some((p, i) => i > 0 && Math.max(p.x, pts[i-1].x) > b.x + 1 && Math.min(p.x, pts[i-1].x) < b.x + b.w - 1
                                                     && Math.max(p.y, pts[i-1].y) > b.y + 1 && Math.min(p.y, pts[i-1].y) < b.y + b.h - 1);
  const out = {};
  // (a) a wide box between source and target: routed around it
  const boxes = new Map([["s", {x: 0, y: 0, w: 100, h: 40}], ["m", {x: -20, y: 100, w: 240, h: 40}], ["t", {x: 0, y: 200, w: 100, h: 40}]]);
  const a = route_edges(A, boxes, [{key: "k", from: "s", out_x: 50, to: "t"}], 24).get("k");
  out.a = {pts: a, orth: orth(a), start: a[0], end: a[a.length - 1], crosses: crosses(a, boxes.get("m"))};
  // (b) [RF 3] two edges into one box: both orthogonal, both at its one entry point
  const b2 = new Map([["s1", {x: 0, y: 0, w: 100, h: 40}], ["s2", {x: 300, y: 0, w: 100, h: 40}], ["t", {x: 150, y: 200, w: 100, h: 40}]]);
  const rb = route_edges(A, b2, [{key: "k1", from: "s1", out_x: 50, to: "t"}, {key: "k2", from: "s2", out_x: 50, to: "t"}], 24);
  out.b = [...rb.values()].map(p => ({orth: orth(p), end: p[p.length - 1]}));
  // (c) [RF 4] an edge from a box to itself: no throw, orthogonal
  try { const c = route_edges(A, new Map([["s", {x: 0, y: 0, w: 100, h: 40}]]), [{key: "k", from: "s", out_x: 50, to: "s"}], 24).get("k");
        out.c = {orth: orth(c), n: c.length}; } catch (e) { out.c = {threw: String(e)}; }
  // (d) nothing to route
  out.d = route_edges(A, boxes, [], 24).size;
  // (e) still alive after (c): the wasm did not abort
  out.e = route_edges(A, boxes, [{key: "k", from: "s", out_x: 50, to: "t"}], 24).get("k").length;
  return out; })()`);
console.log("   ", JSON.stringify(r));
check(r.a.orth && !r.a.crosses, "(a) orthogonal, around the box in between");
check(r.a.start.x === 50 && r.a.start.y === 40, "(a) leaves the source's bottom edge at its out-port: " + JSON.stringify(r.a.start));
check(r.a.end.x === 24 && r.a.end.y === 200, "(a) enters the target's top edge at in_x: " + JSON.stringify(r.a.end));
check(r.b.length === 2 && r.b.every(x => x.orth && x.end.x === 174 && x.end.y === 200), "[RF 3] (b) two edges into one box: orthogonal, one entry point");
check(r.c && !r.c.threw && r.c.orth, "[RF 4] (c) an edge from a box to itself: " + JSON.stringify(r.c));
check(r.d === 0, "(d) no edges: an empty map");
check(r.e >= 2, "(e) the router still works after (c)");
// <Test ending>
```

- [ ] **Step 2: Append `route_edges` to `all_tests`, build, and run it**

Run: `cmake --build .build --target websock_ex_introspect -j16 && xo-websock/example/introspect/test/run.sh .build/xo-websock/example/introspect/websock_ex_introspect route_edges`
Expected: FAIL with `EXC ... load_router is not defined`.

- [ ] **Step 3: Implement `route_edges.js`** per the Interfaces block and the Global Constraints' libavoid traps:
  - one `ShapeRef` per box;
  - one down out-pin per edge at `(out_x, h)` on its source, with a fresh class id;
  - one up in-pin per target box at `(in_x, 0)`, shared by every edge into that box, `setExclusive(false)`;
  - one `ConnRef` per edge, then `processTransaction()`;
  - read each `displayRoute()`, then free the router.

  **Finding the free call is part of this step.** The spike never did it.  Look in the module's typings (`dist/index.d.ts` in the npm package, e.g. `npm pack libavoid-js@0.4.5` into the job's scratch directory) for `destroy`, or an embind `delete()` on `Router`.  Ledger which call it is, as a Ruling.

- [ ] **Step 4: Run it**

Run: the Step 2 command.
Expected: `route_edges: ok`.  If (c) throws or aborts the wasm, a self-edge needs its own handling: e.g. route it as a fixed loop out of the bottom, down, left and up, around the box's left side, without libavoid.  Ledger that as a Ruling, and make (c) pass.

- [ ] **Step 5: Check the router is freed.** Run `route_edges` on case (a)'s boxes 2000 times in one `ev(...)`, and compare the wasm memory size before and after.  Find the heap: the instance or its module may expose `HEAP8` / `wasmMemory`; ledger which.  Expected: no growth beyond the first call's.  If the memory can't be reached from the instance, ledger that as a Ruling, and make sure the free call is at least reached on every path, by reading the code.

- [ ] **Step 6: Run the whole suite**

Expected: `23 passed, 1 failed` (`hysteresis` still fails: nothing uses `route_edges` yet).

- [ ] **Step 7: Stop for RC to review and commit.**

---

### Task 3: `place_boxes` by ownership, wired to `route_edges`

**Files:**
- Modify: `xo-websock/example/introspect/mount-origin/introspect.js`:
  - `draw_aux`, from `:1696` ("boxes a member edge arrives at") through step 4 (`:1804-1817`);
  - `:160` (`const elk`), for the router load beside it;
  - `out_ports` (`:744-756`) keeps deciding port geometry; its output now goes to `route_edges` instead of ELK.
- Create: `xo-websock/example/introspect/test/avoid_load.mjs`
- Modify: `xo-websock/example/introspect/test/run.sh` (append `avoid_load`)

**Interfaces:**
- Consumes: `load_router()`, `route_edges(avoid, boxes, edges, in_x)` (Task 2); `out_ports(d, out)`, `in_port_x`, `is_ownership(kind)`, `elk_children(nodes, f)` (existing).
- Produces: `place_boxes(nodes, edges) -> Promise<{at, groups, size, own_routes}>`:
  - `at`: Map(box or group id → `{x, y}`), absolute, without `pad` (what step 3 builds today);
  - `groups`: as `draw_groups` takes them today;
  - `size`: `{w, h}` of the laid-out drawing, without `pad`;
  - `own_routes`: Map(edge key → absolute points), for every ownership edge ELK routed.  Step 4 draws a "drawn" (wanted-fallback) ownership edge from it, exactly as today (spec, "What each stage sees").  The spec names `{at, groups, size}` and is silent on the fallback edges' routes; `own_routes` carries them.  The key matches `routed`'s (`${e.kind}:${e.source}>${e.target}`).
- Invariants:
  - ELK's graph contains ownership edges only, and no port of any kind (`grep -n '#in' introspect.js` finds no ELK port once this task is done);
  - ELK keeps every other option it has today (`:1701-1726`).  `MODEL_ORDER` cycle breaking and the priority option become moot without member edges; leave them in, and add one comment saying so;
  - the router instance comes from one `load_router()` call, started at page load (`router = load_router()` beside `elk`), and awaited inside `draw_aux` before any layout work.  On rejection: `#status` reads `error: edge router (libavoid) failed to load: <message>`, and `draw_aux` returns without drawing;
  - `route_edges` runs synchronously after `place_boxes` resolves AND after the `seq !== draw_seq` check, so a superseded draw never routes.

- [ ] **Step 1: [RF 5] Look for an ownership-isolated box.** Read how `draw_aux` builds the shown `nodes` and the `edges` it gives ELK (`introspect.js:1350-1695`), and the visibility rules (`:338-380`).  Decide whether a shown box can lack any ownership edge to another shown box (e.g. a box whose owner is hidden).  Then confirm it in the browser: in the default view, after Show all, and after Hide all, list the shown box ids that no ownership edge in that draw touches, other than the Webserver root.  To capture the edges, add a temporary `console.log` in `draw_aux` that you delete before Step 4, or call the same helpers from an `ev(...)`.  Ledger what you find.  If such a box exists, check `hysteresis` and the spike screenshots for how ELK packs it.  Leave ELK's component handling at its default unless a test fails, and ledger it.

- [ ] **Step 2: Write `avoid_load.mjs`** ([RF 1], and the failure the spec accepts)

This test does NOT use the standard preamble's first-page wait: it must intercept the module before the page asks for it.

```js
// libavoid loading (.xo-backlog/introspect-layout/spec.md, "Failure"):
// blocked -> a visible error and no drawing; slow -> the first draw waits
import fs from "node:fs";
const [,, cdp_port, port, out] = process.argv;
const sleep = (ms) => new Promise(r => setTimeout(r, ms));
let ok = true; const check = (c, msg) => { console.log(`${c ? "ok  " : "FAIL"} ${msg}`); if (!c) ok = false; };
async function page(mode) {   // mode: "block" | "delay"
  const tgt = await (await fetch(`http://localhost:${cdp_port}/json/new?about:blank`, {method: "PUT"})).json();
  const ws = new WebSocket(tgt.webSocketDebuggerUrl); await new Promise(r => ws.onopen = r);
  let seq = 0; const pending = new Map();
  const call = (method, params = {}) => new Promise(r => { const id = ++seq; pending.set(id, r); ws.send(JSON.stringify({id, method, params})); });
  ws.onmessage = async (e) => { const m = JSON.parse(e.data);
    if (m.id && pending.has(m.id)) { pending.get(m.id)(m); pending.delete(m.id); return; }
    if (m.method === "Fetch.requestPaused") {
      const rid = m.params.requestId;
      if (mode === "block") call("Fetch.failRequest", {requestId: rid, errorReason: "BlockedByClient"});
      else { await sleep(3000); call("Fetch.continueRequest", {requestId: rid}); } } };
  const ev = async (expr) => { const r = await call("Runtime.evaluate", {expression: expr, returnByValue: true, awaitPromise: true}); return r.result.result?.value; };
  await call("Fetch.enable", {patterns: [{urlPattern: "*libavoid*"}]});
  await call("Page.navigate", {url: `http://localhost:${port}/`});
  return {ws, call, ev};
}
{ // blocked
  const {ws, ev} = await page("block");
  let st = "";
  for (let i = 0; i < 100 && !/libavoid/.test(st); i++) { await sleep(100); st = await ev(`document.getElementById("status").textContent`) || ""; }
  check(/^error: edge router \(libavoid\) failed to load/.test(st), "blocked: the status line says so: " + JSON.stringify(st));
  check(await ev(`document.querySelectorAll("g.node").length`) === 0, "blocked: nothing drawn");
  ws.close();
}
{ // [RF 1] slow: the snapshot arrives first; the draw waits, then routes
  const {ws, call, ev} = await page("delay");
  for (let i = 0; i < 150 && !(await ev(`typeof last_event !== "undefined" && !!last_event && document.querySelectorAll("g.node").length >= 1`)); i++) await sleep(100);
  await ev(`settled()`);
  await ev(`document.getElementById("show-all").click(); 1`); await ev(`settled()`);
  check(!/error/.test(await ev(`document.getElementById("status").textContent`)), "slow: no error");
  check(await ev(`document.querySelectorAll("path.edge.member").length`) >= 10, "slow: member edges drawn");
  fs.writeFileSync(out, Buffer.from((await call("Page.captureScreenshot", {format: "png"})).result.data, "base64"));
  ws.close();
}
console.log(ok ? "ALL OK" : "SOME FAILED");
process.exit(ok ? 0 : 1);
```

- [ ] **Step 3: Append `avoid_load` to `all_tests`, build, and run `avoid_load hysteresis`**

Expected: `avoid_load: FAIL` (the page never loads libavoid, so a block changes nothing and the status never names it) and `hysteresis: FAIL`.

- [ ] **Step 4: Implement**, per the Interfaces block:
  1. `router = load_router()` at page load; awaited in `draw_aux`, with the failure shown as the invariant says.
  2. `place_boxes(nodes, edges)`: today's ELK graph, built from ownership edges only and with no ports.  It returns `at` (group children made absolute, as `:1757-1767` does), `groups`, `size`, and `own_routes` (sections offset by their container, as `:1806-1816` does).
  3. In `draw_aux`, step 3 places the boxes from `place_boxes`' result.  Then route the member edges:
     - build `boxes` from the placed boxes (`d.x - pad`, `d.y - pad`, `d.w`, `d.h`: the frame `pts` use);
     - build `edges` from the drawn member edges: `out_x` is the port position `out_ports` gives for that edge on its source box;
     - call `route_edges(avoid, boxes, edges, in_port_x)`.
  4. Step 4's `routed` takes a member edge's points from `route_edges` and a drawn ownership edge's from `own_routes`.  Everything after `routed` is unchanged.
  5. Delete what nothing uses any more: the `member_in` set, ELK port lists, the `#in` target rewriting, `laid_edge`.

- [ ] **Step 5: Run `avoid_load hysteresis routes route_edges`**

Expected: all four `ok`.  `hysteresis` passing is the point of the spec.  If it still fails, print `moved`: boxes that still move point at something besides member edges steering ELK.  Do not tune ELK options to pass it.  Stop and report the numbers.

- [ ] **Step 6: Run the whole suite**

Expected: `25 passed, 0 failed`.  Pay attention to the three "server above its sessions" checks (`endpoint_expand.mjs:46`, `sender_expand.mjs:41`, `sesstable_expand.mjs:52`) and `edge_kinds.mjs`.  An existing test that read ELK-specific geometry and now fails gets a ledgered Ruling (the test pinned ELK, not behaviour), stated in the stop report.  Never a silent edit.

- [ ] **Step 7: Screenshots for RC.** Copy `hysteresis.png` and `routes.png` from the run's temp directory to `~/.claude/jobs/<job>/tmp/` (or another path you name in the report), so RC can look at route quality.

- [ ] **Step 8: Stop for RC to review and commit.**

---

### Task 4: Tune, measure the unpromised case, close out

**Files:**
- Modify: `xo-websock/example/introspect/mount-origin/route_edges.js` (spacing constants)
- Modify: `xo-websock/example/introspect/mount-origin/introspect.js` (`elk.layered.spacing.nodeNodeBetweenLayers` only, if tuning calls for it)
- Modify (backlog, commit and push allowed): `.xo-backlog/introspect-layout/spec.md` (Status, and measured results under "Not promised")

**Interfaces:**
- Consumes: everything above.  Produces: no new names.

- [ ] **Step 1: Tuning candidates, with screenshots.** Using the `routes` test's Show-all state, capture one screenshot per setting:
  - `idealNudgingDistance` ∈ {6, 10};
  - `shapeBufferDistance` ∈ {8, 12};
  - `nodeNodeBetweenLayers` ∈ {50, 70}.

  ELK no longer reserves room for member edges between layers, which is why that one is in the list.  That's eight screenshots, each named by its setting.  `routes` and `hysteresis` must pass under every candidate you show RC.  Present them to RC.  **RC picks; do not pick.**  Stop here until RC answers, then set the chosen values.

- [ ] **Step 2: Measure the case the spec does not promise.** Write the measurement as a throwaway script in the job's scratch directory (not a repo test): `hysteresis.mjs`'s method, but the click opens a box that adds OWNED children.  One run for the session table (`toggle_children` on the session table box), one for a session.  Record the `moved` / `swaps` numbers, with the date and the box opened, in the spec's "Not promised" section.  Rule 6 applies: keep the "Unverified" sentence, and add the measurement after it.  If either run moves boxes, write a backlog ticket for option A behind `place_boxes` (`.xo-backlog/xo-websock/issues/NN-introspect-layout-order-preservation.md`, NN the next free number), citing the numbers.

- [ ] **Step 3: Full verification**
  - Run: `cmake --build .build --target websock_ex_introspect -j16 && xo-websock/example/introspect/test/run.sh .build/xo-websock/example/introspect/websock_ex_introspect`
    Expected: `25 passed, 0 failed`.
  - Run: `xo-build --sweep`
    Expected: the closing line `xo-build: --sweep ok (build and utest)`, with totals as in `.xo-backlog/CONVENTIONS.md` (`73 attempted: 73 ok`; `47 ok, 26 with no tests`).  A different total gets explained, not ignored (CONVENTIONS, "Verifying a change").

- [ ] **Step 4: Close the spec.** Set `Status:` to `done <date> (umbrella <commits>, once RC commits)`, and tick its "Done when" items against the evidence above.  Commit and push the backlog.

- [ ] **Step 5: Stop for RC to review and commit the umbrella changes.**
