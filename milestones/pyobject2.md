# pyobject2 — faceted objects in python

Status: open
Type: milestone

Let python construct, hold, render and release fomo objects from `xo-object2`,
as a harness for driving and inspecting the c++ object model.

Design: `.xo-backlog/pyobject2/spec.md`.

## Why this shape

The plan of record was to extend `genfacet` with a per-facet handle class
`H<Foo>` for python to bind against. It was dropped: with `DRepr` known
statically in a pybind TU, `obj<AFacet,DRepr>` is constructible from a bare data
pointer with no registry lookup, so `H<Foo>` would be a forwarding layer beneath
pybind's own forwarding layer. What repeats is per *facet*, so a hand-written
binder template per facet replaces it — O(facets + reprs) instead of
O(facets x reprs), and no generator to maintain.

## Scope

v1 is static: one python class per representation, facets rotated at compile
time inside the binders. Collection and any erased `Obj` type are v2, gated on
lifting `FacetRegistry` rotation into python.

## Progress

Tickets carrying `Milestone: pyobject2`; `xo-sdlc --milestones` counts them.
