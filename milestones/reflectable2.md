# reflectable2 — fomo objects participate in reflection

Status: open
Type: milestone

Make faceted objects (`obj<AFacet,DRepr>`) reflectable, so c++ code can
interrogate them at runtime without knowing their representation. First
consumer is `xo-printjson`.

Design: `.xo-backlog/reflectable2/spec.md`.

## Why this shape

`xo-reflect` is NOT modified. `TypeDescrExtra` and `EstablishTdx` are public
extension points, so the fomo-aware pieces live in a new subsystem above
reflect. That matters: hosting them in reflect would create a reflect -> facet
edge, and reflect has far more dependents than facet does. The means of
checking that cost:

```bash
for s in $(xo-deps --users-of=xo-reflect --format=names -q); do
    xo-deps --why=$s:xo-facet -q >/dev/null || echo "$s"
done | wc -l    # subsystems that would newly acquire xo-facet
```

Capability is OPT-IN, per type, following `xo::reflect::SelfTagging` — which
solves the same get-the-real-type-at-runtime problem for non-fomo objects, and
which `xo-printjson` already consumes as an opt-in interface
(`print_obj(rp<SelfTagging>)`).

## The longer-range reason

JSON for fomo objects is the near-term deliverable, not the point. With
printjson and reflect working over fomo, xo state becomes reachable from a
browser — so visualizations and animations can be grounded in ACTUAL xo state
rather than a parallel model of it maintained by hand. Parallel models drift;
that is the cost this is meant to avoid.

## Progress

Tickets carrying `Milestone: reflectable2`; `xo-sdlc --milestones` counts them.
