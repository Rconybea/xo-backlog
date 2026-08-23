# 01 — AbstractEventProcessor should inherit ref::Displayable, not ref::Refcount

Status: open — DEFERRED, probably permanently (see disposition)
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

## The real obstacle: EventStoreImpl reaches Refcount by two paths

Measured 2026-08-23. `AbstractEventProcessor`'s `virtual public ref::Refcount`
is **load-bearing**, and not for the reason it looks like:

```bash
grep -n 'class EventStoreImpl' -A3 xo-reactor/include/xo/reactor/EventStore.hpp
#   class EventStoreImpl : public SinkEndpoint<Event>,
#                          public AbstractEventStore,
#                          ReducerBase<Event, EventTimeFn>
grep -n 'class AbstractEventStore' xo-reactor/include/xo/reactor/EventStore.hpp
#   class AbstractEventStore : virtual public ref::Refcount
```

So:

```
EventStoreImpl -> SinkEndpoint -> .. -> AbstractSink
                                     -> virtual AbstractEventProcessor
                                     -> virtual public ref::Refcount    (path 1)
               -> AbstractEventStore -> virtual public ref::Refcount    (path 2)
```

Path 2 does **not** go through AbstractEventProcessor. So AEP's virtual
inheritance of Refcount is not made redundant by
`AbstractSink : public virtual AbstractEventProcessor` — both paths must be
virtual for EventStoreImpl to have ONE Refcount subobject.

`Displayable : public Refcount` is non-virtual. Inheriting it non-virtually from
AEP therefore gives EventStoreImpl a non-virtual Refcount on path 1 and a
virtual one on path 2: **two subobjects, two reference counts on one object**,
and ambiguous conversions to `Refcount*`. For an intrusively refcounted type
that is a lifetime bug, not merely a compile error.

Three ways out, none free:

1. **`Displayable : virtual public Refcount`** — smallest edit, preserves the
   lattice. Costs every Displayable a virtual base; xo-refcnt is build position
   15, so the cost is felt widely.
2. **`AbstractEventProcessor : virtual public ref::Displayable`** — keeps the
   virtualness where it already is, at the cost of more virtual inheritance.
3. **Untangle EventStoreImpl** so `AbstractEventStore` does not independently
   inherit Refcount. Then path 2 disappears and non-virtual works. Much larger.

## Disposition (RC, 2026-08-23): deferred, expect to carry to xo-reactor2

RC's read is that xo-reactor's multiple inheritance was a mistake in hindsight,
and that **this refactor will probably never land in xo-reactor** — the fix
worth having is option 3, and it is not worth doing to a subsystem that a
future `xo-reactor2` would replace. Non-virtual `AEP <- Displayable` is the
design RC wants; xo-reactor cannot have it without option 3.

**So treat this ticket as a design record for xo-reactor2, plus a live hazard
notice for xo-reactor.** The hazard below does not go away by deferring.

## THE HAZARD STAYS LIVE

Because the fix is deferred, the tree keeps **three** constrained partial
specializations of `Prettifier<T>`, disjoint only by accident:

```bash
grep -rn 'requires std::derived_from' --include=*.hpp xo-*/ | grep -v '/\.build/'
#   xo-refcnt/../Displayable.hpp                  derived_from<ref::Displayable>
#   xo-reactor/../AbstractEventProcessor.hpp      derived_from<reactor::AbstractEventProcessor>
#   xo-reactor/../Reactor.hpp                     derived_from<reactor::Reactor>
```

Deriving any class from two of the three makes both constraints match, neither
more specialized: **ambiguous partial specialization**, a hard error whose
message names neither `Displayable` nor `Reactor`. The most likely trigger is
exactly the edit this ticket describes — giving an event processor a
`display_string()` by inheriting Displayable. Anyone who tries it should read
this ticket first.

## Done when — IF it is ever done (xo-reactor2, most likely)

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
