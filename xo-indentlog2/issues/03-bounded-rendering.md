# 03 — bounded rendering: max-lines / max-items / max-depth, and cycle detection

Status: open
Type: feature

A `PrettySink` will render whatever it is given, however large or however
deeply nested, and it will not terminate on a cyclic object graph.

**Proposed by RC 2026-08-09**, arising from the `DList` conversion in
`.xo-backlog/xo-printable2/issues/01`: pretty-printer tropes like max-lines,
max-items and max-depth, with **the ellipsis well placed** rather than
replacing the whole rendering — plus cycle detection.

## What the legacy printer did, and why it is not the answer

`DList::pretty_deprecated` could only manage this:

```cpp
if (ppii.upto()) {
    /* try to fit on one line: write elements */
} else {
    pps->write("(...)");     /* did not fit */
    return false;
}
```

Pinned until 2026-08-09 by `xo-gc/utest/Object2.test.cpp`: a 20-element list at
margin 80 rendered as exactly `(...)`.

That is not bounded rendering, it is *abandoned* rendering — the two-pass
protocol asked "does it fit?", was told no, and had nowhere to put the
elements. Every element is lost, including the first few that would have fit.

`DList::pretty` now renders structurally instead, and a 20-element list breaks
across 20 lines. **That is better, and it is also unbounded**: a million-element
list produces a million lines, and a cyclic list never returns.

## What is wanted instead

Bounds enforced by the sink, so no printer has to implement them and every
printer benefits:

- **max-items** — `(1 2 3 ... 998 999 1000)` or `(1 2 3 ...)`; the elision is
  positioned within the structure rather than replacing it
- **max-lines** — stop after N lines with a trailing indication
- **max-depth** — elide below a nesting level, `(1 (2 (...)))`
- **cycle detection** — a back-reference marker rather than non-termination

The point RC made: the ellipsis should be *well placed*. `(...)` throws away
everything; `(1 2 3 ...)` keeps what was affordable and says so.

## Cycle detection is a correctness gap today, not just a nicety

Neither protocol has it:

- `pretty_deprecated` walks the list in its `upto` pass and would loop forever
  on a cycle
- `DList::pretty` walks it in the single pass and does the same

So this is **not a regression introduced by the migration** — it is pre-existing
and now inherited. Worth stating plainly because the conversion is otherwise
behaviour-preserving, and someone comparing the two will find they are equally
broken here.

The object model makes cycles easy to construct: `DList` holds `head_`/`rest_`
as GC-visited children, and nothing prevents a list containing itself.

## Design questions

- **Where do bounds live?** `PpConfig` is the natural home, alongside the
  margins — which would make them settable through
  `.xo-backlog/xo-indentlog2/issues/02`'s `PrettyContext`. That is the main
  argument for doing 02 first.
- **What does a printer have to do to cooperate?** Ideally nothing: if the sink
  counts items between `begin()`/`end()` and tracks nesting, printers stay
  unaware. Worth confirming that holds for a printer emitting `put()` directly
  rather than through `pretty_struct`.
- **Cycle detection needs identity.** For facet objects that is the data
  pointer; a `std::set<const void *>` on the sink for the duration of a render
  is the obvious first cut, though it costs an allocation per render.
- **Is elision output-visible in a way that breaks round-tripping?** For
  Schematika values, an elided rendering must not read back as a shorter value
  — see `.xo-backlog/xo-object/issues/01`, where round-tripping is a live
  question.

**Files:**
- `xo-indentlog2/include/xo/indentlog2/print/PpConfig.hpp` — where bounds
  would be configured
- `xo-indentlog2/include/xo/indentlog2/print/PrettySink.hpp`,
  `PpState.hpp` — where they would be enforced
- `xo-object2/src/object2/DList.cpp` — the motivating printer, unbounded today

**Done when:**
- a rendering can be bounded without the printer knowing about it
- a cyclic object graph renders in finite time with a visible marker
- bounds are reachable from configuration, not hardcoded
- the elision states what was elided rather than discarding the rendering

## Notes

Do not let this block `.xo-backlog/xo-printable2/issues/01` phase C. Structural
rendering without bounds is strictly better than `(...)` — it loses nothing
that the legacy form kept — so converting printers now and bounding later is
the right order. This ticket exists so the gap is recorded rather than
discovered by a large or cyclic list in production.
