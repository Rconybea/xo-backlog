# 01 — AbstractEventProcessor should inherit ref::Displayable, not ref::Refcount

Status: open
Type: refactor
Milestone: ostream-containment

**Sequencing: land the current xo-reactor pretty() refactor first.** This
ticket removes duplication that refactor creates; doing both at once would mix
a mechanical conversion with a hierarchy change.

## Why

`Prettifier<T>` now has THREE constrained partial specializations in the tree:

```bash
grep -rn -B2 'struct Prettifier<T>' --include=*.hpp xo-*/ | grep -v '/\.build/' | grep derived_from
#   xo-refcnt/include/xo/refcnt/Displayable.hpp          derived_from<ref::Displayable>
#   xo-reactor/include/xo/reactor/AbstractEventProcessor.hpp  derived_from<reactor::AbstractEventProcessor>
#   xo-reactor/include/xo/reactor/Reactor.hpp            derived_from<reactor::Reactor>
```

They are disjoint **today** — verified 2026-08-23:

```bash
grep -n 'class AbstractEventProcessor' xo-reactor/include/xo/reactor/AbstractEventProcessor.hpp
#   class AbstractEventProcessor : virtual public ref::Refcount   <- NOT Displayable
grep -n 'class Reactor :' xo-reactor/include/xo/reactor/Reactor.hpp
#   class Reactor : public ref::Refcount                          <- NOT Displayable, not an AEP
```

Nothing enforces that, and nothing records it. **If any class ever derives from
two of the three, both constraints match, neither is more specialized, and it is
an ambiguous partial specialization** — a hard error at the point of use whose
message will not mention `Displayable` or `Reactor` at all. `Refcount` is the
common base and `Displayable : public Refcount`, so the collision is one
plausible edit away: giving `AbstractEventProcessor` a `display_string()` by
inheriting `Displayable` trips it immediately.

## Proposal

`ref::Displayable` already declares exactly what the reactor refactor added:

```cpp
/* xo-refcnt/include/xo/refcnt/Displayable.hpp:19 */
virtual void pretty(PpSink & pp) const = 0;
```

So the reactor-local declarations are duplicates, not peers. Make
`AbstractEventProcessor` (and `Reactor`) inherit `ref::Displayable`, then
**delete** from xo-reactor:

- the `virtual void pretty(PpSink&) const = 0;` declarations on both roots
- both constrained `Prettifier<T>` specializations

leaving `Displayable.hpp`'s single specialization to cover the whole hierarchy.
That is also what its own comment asks for: *"Must live here to insure that it's
consistently applied (else ODR violation!)"* — three overlapping definitions of
the same idea is the situation that comment is trying to prevent.

## `display_string()` is NOT a complication — corrected 2026-08-23

This ticket first flagged a purity conflict: `Displayable.hpp:21` declares
`display_string()` `= 0` while `AbstractEventProcessor.hpp:44` declares it
non-pure and defines it as `tostr(*this)`.

**That is the intended arrangement, not a clash.** RC: Displayable *cannot*
implement it, because `tostr()` lives in xo-indentlog2 and xo-refcnt does not
depend on it — verified 2026-08-23:

```bash
xo-deps --deps-of=xo-refcnt --format=names
#   xo-ppsink  xo-refcnt  xo-reflectutil  xo-timeutil       <- no xo-indentlog2
grep -rn indentlog2 xo-refcnt/ --include=*.hpp --include=*.cpp | grep -v '/\.build/'
#   Displayable.hpp:20: // implement display_string() in derived classes that also have xo-indentlog2
```

So the pure virtual states a contract that only subsystems carrying
xo-indentlog2 can satisfy, and AEP's definition satisfies it. Nothing to
reconcile. Recorded rather than deleted per rule 6: the conflict reading is
plausible from the two declarations alone, and the refuting fact is a
dependency edge, not something visible in either file.

## One real complication

**Virtual vs non-virtual base.** `AbstractEventProcessor : virtual public
   ref::Refcount` but `Displayable : public Refcount` (non-virtual). Swapping in
   Displayable changes the inheritance shape for every event processor, and
   several use multiple inheritance —
`KalmanFilterSvc : Sink1<..>, DirectSourcePtr<..>` is the sharpest case.
Whether Displayable needs `virtual public Refcount` is the real — and only —
design question in this ticket.

## Done when

- `AbstractEventProcessor` and `Reactor` inherit `ref::Displayable`
- xo-reactor declares no `pretty()` pure virtual and no `Prettifier<T>` of its own
- exactly one constrained `Prettifier<T>` remains in the tree:

```bash
grep -rn 'requires std::derived_from' --include=*.hpp xo-*/ | grep -v '/\.build/' | grep -c Prettifier
#   expect 1
```

- `xo-build --sweep` green

## See also

`.xo-backlog/xo-refcnt/issues/03-prettifier-visibility-odr.md` — the open policy
question about where a `Prettifier` may be declared. This ticket is the same
question's other half: not *where* one lives, but how many may claim one type.
