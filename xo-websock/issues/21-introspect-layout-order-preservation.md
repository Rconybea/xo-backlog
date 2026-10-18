# 21 -- introspect layout: showing / hiding owned children still reorders boxes

Status: open
Type: bug (UX)
Raised: 2026-10-17, from `.xo-backlog/introspect-layout/spec.md` ("Not promised")

`introspect-layout` stopped MEMBER edges from moving boxes: ELK now places
boxes by ownership edges alone (`place_boxes`, umbrella `4e046512`).
Opening a box that only grows (IntrospectReceiver) no longer moves anything
(`xo-websock/example/introspect/test/hysteresis.mjs`).

Showing or hiding a box's OWNED children still changes what ELK is given,
and ELK still answers with a different arrangement.

## Measured (2026-10-17, umbrella `4e046512`, fresh server, one session)

Starting from Show all, I hid a box's owned children with `toggle_children`
and showed them again.  For the boxes present in both views I removed the
shift they all share, then counted boxes that moved more than 20px, and
same-row pairs that swapped left/right:

| children toggled | moved | swaps |
|---|---|---|
| `server/session_table`'s 5 | 5 of 9 | 3: `url_router`, `ws_config`, `session_table` (the server's nested boxes) change order |
| `session:1`'s 4 | 3 of 10 | 0 |

The script was throwaway: it was `hysteresis.mjs` with its click replaced by
two `toggle_children(<box>)` calls.

## Direction

Option A from the spec: keep the previous arrangement and let ELK change
it as little as possible.  It goes behind `place_boxes(nodes, edges)`, the
seam `introspect-layout` left for it, and nothing else changes.

Candidates, cheapest first (unverified, each needs a measurement like the
one above):

1. **Model order for siblings.**  The swaps above are siblings inside one
   group trading places.  ELK layered has options to respect input order
   during crossing minimisation (`elk.layered.considerModelOrder.strategy`,
   e.g. `NODES_AND_EDGES`).  `layout()` lists boxes in a stable order, so
   siblings would keep their order.  That would not stop moves from boxes
   changing size or count.
2. **Interactive ELK, seeded from `drawn_at`.**  The spike tried this with
   member edges still present: 5 of 14 moved, and the drawing lurched 883px
   vertically.  It has not been retried with ownership-only placement.
3. **Incremental layout outside ELK.**  MSAGL's incremental mode
   (`msagljs`), or a commercial engine (yFiles).  Last resort.

**Done when:** a test in the style of `hysteresis.mjs` toggles the session
table's children and passes (no box moves beyond the shared shift, no
swaps).  The existing browser suite stays green.
