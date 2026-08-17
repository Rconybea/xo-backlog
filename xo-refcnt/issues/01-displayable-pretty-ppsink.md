# 01 — Displayable::display(std::ostream&) -> Displayable::pretty(PpSink&)

Status: fixed 2026-08-16
Type: task
Milestone: ppsink-migration
Progress: grep -rn 'virtual void display(std::ostream' xo-refcnt/ xo-websock/ --include=*.hpp 2>/dev/null | grep -v '/\.build/' | wc -l

Raised by RC 2026-08-16. `xo::ref::Displayable`
(`xo-refcnt/include/xo/refcnt/Displayable.hpp:12`) still declares

```cpp
virtual void display(std::ostream & os) const = 0;
```

which is the last ostream-shaped rendering interface in the refcnt layer. It
should become `virtual void pretty(xo::pp::PpSink & sink) const`.

## The scope is two classes, not a hundred

A grep for `display(std::ostream` across the tree returns ~100 declarations in
~30 subsystems, which badly overstates this ticket. Almost all of them are
independent `display()` methods hanging off `ref::Refcount` (or off nothing),
not overrides of `Displayable`. The actual hierarchy:

```bash
grep -rn 'public.*Displayable' --include=*.hpp --include=*.cpp xo-*/ \
  | grep -v '/\.build/' | grep -v 'xo-indentlog/'
#   xo-websock/include/xo/websock/Webserver.hpp:76:  class Webserver : public ref::Displayable
```

**One subclass.** So the change is: the base declaration, `Webserver`'s
override (`xo-websock/src/websock/Webserver.cpp:1977`), `display_string()`, and
the inline inserter in the header.

Re-derive that before starting rather than trusting it — `GeneralizedExpression`,
`Reactor` and `AbstractEventProcessor` each declare their own pure-virtual
`display(std::ostream&)` and look like members of this hierarchy in a grep, but
all three derive from `ref::Refcount` directly.

## Precedent: this was already done, one layer up

`.xo-backlog/xo-alloc/issues/01-object-pretty-ppsink.md` (fixed 2026-08-08) did
exactly this for `xo::gc::Object`: `virtual void display(std::ostream&)` became
`virtual void pretty(xo::pp::PpSink&)`, and the inserter moved to an opt-in
`xo-alloc/include/xo/alloc/alloc_ostream.hpp` following
`xo-webutil/include/xo/webutil/webutil_ostream.hpp`.

Worth reading first. In particular it records the bridging strategy for a
hierarchy split across subrepos — carry both signatures for a few commits,
with the *base* `display()` never delegating, so nothing recurses. With one
subclass here that may be unnecessary, but xo-refcnt and xo-websock are separate
subrepos, so a flag-day commit still puts a non-building commit into xo-refcnt's
own history if the two are pushed independently.

## Why it is live now

`Displayable::display_string()` is

```cpp
/* xo-refcnt/src/Displayable.cpp:11 */
return tostr0(*this);
```

and `Displayable` has no `Prettifier`, so it renders through pretty()'s
`operator<<` fallback. Since `45fd03bc` made that fallback opt-in per TU, this
line is **the first thing in the tree that fails to compile** — it is what stops
`cmake --build .build` at xo-refcnt, before anything else can be measured
(`.xo-backlog/xo-ppsink/issues/12-operator-fallback-inventory.md` records the
same line blocking the first inventory attempt).

Converting to `pretty()` fixes it at the root rather than by adding an include.

## Shape

- `Displayable.hpp:12` — `virtual void pretty(xo::pp::PpSink &) const = 0`.
  Forward-declare `PpSink`; do not include `PpSink.hpp` here if it can be
  avoided (`Object.hpp` took that care for the same reason: it sits at the root
  of an object model and should stay cheap). Unverified whether `Refcounted.hpp`
  already drags it in.
- `Displayable.cpp:11` — `display_string()` becomes `tostr(*this)` over a
  `Prettifier<Displayable>`, or `pps.pp(*this)` directly.
- `Displayable.hpp:20-24` — the inline `operator<<(std::ostream&, Displayable
  const&)` moves to a bridge header, per `ostream-containment`.  **Which one is
  a decision that belongs to `issues/02`**, which moves the `rp<T>`/`Borrow<T>`
  inserters out of `Refcounted.hpp`: either they share one
  `Refcounted_ostream.hpp`, or this gets its own `Displayable_ostream.hpp`
  pairing with `Displayable.hpp` the way `Refcounted_pp.hpp` pairs with
  `Refcounted.hpp`.  Do not invent a third spelling.
- `xo-websock` — `Webserver::display` -> `pretty`, and whatever
  `Webserver.hpp:111` declares.

## Done when

- `Displayable` names no `std::ostream`
- `xo-build --sweep` reads
  `62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`
- `nix-build ci.nix -A xo-websock --no-out-link` (xo-refcnt's public header
  surface changes, so a consumer is the check — see
  `.xo-backlog/tostr-arena/issues/01`)

## Explicitly out of scope

The ~100 other `display(std::ostream&)` methods. They form several unrelated
families — xo-kalmanfilter (14), xo-expression (12), xo-reactor (9),
xo-alloc's `GcStatistics`/`ObjectStatistics` (already excluded by
`xo-alloc/issues/01` as non-virtual and outside the hierarchy), xo-reflect,
xo-simulator, xo-process, xo-websock, xo-ordinaltree — and each wants its own
decision. Converting them is `ostream-containment`'s business, not this
ticket's.

## Fixed 2026-08-16 (`216cc6b8`)

`Displayable` names no `std::ostream`. The shape landed as:

```cpp
/* xo-refcnt/include/xo/refcnt/Displayable.hpp:14-18 */
using PpSink = xo::pp::PpSink;
virtual void pretty(PpSink & pp) const = 0;
virtual std::string display_string() const = 0;
```

with `xo-websock/src/websock/Webserver.cpp` converted alongside
(`Webserver.hpp:111-112`).

Three ways it differs from the plan above, each deliberate:

- **The inline inserter was deleted, not moved.** So `issues/02`'s naming
  question — `Refcounted_ostream.hpp` vs a paired `Displayable_ostream.hpp` —
  never had to be answered for this header. Nothing streamed a `Displayable`
  directly.
- **`display_string()` survives as pure virtual** rather than being implemented
  once in the base. The old body (`return tostr0(*this)`) is preserved under
  `#ifdef OBSOLETE` in `Displayable.cpp`; the header comment explains why —
  "implement display_string() in derived classes that also have xo-indentlog2",
  i.e. the base cannot call `tostr` without pulling indentlog2 into xo-refcnt,
  which the reserved list in `tostr0.hpp:11-15` forbids.
- **`PpSink.hpp` is included, not forward-declared.** The plan asked for a
  forward declaration to keep the header cheap. It buys nothing here:
  `Displayable.hpp` includes `Refcounted.hpp`, which already includes
  `<xo/ppsink/tostr0.hpp>` (`Refcounted.hpp:6`) for the aliasing-ctor throw at
  `:103-107`. The cost was already paid one level down. Recorded so the next
  reader does not "fix" it.

Follow-up applied at the same time: `Displayable.cpp` still included
`<xo/ppsink/pretty_ostream.hpp>`, whose only user was inside the `#ifdef
OBSOLETE` block. Dropped —, the last of the three stopgap includes flagged in
`.xo-backlog/xo-ppsink/issues/12-operator-fallback-inventory.md`, of which two
are now gone (this and `xo-reactor/include/xo/reactor/Sink.hpp`) and one
(`xo-pyunit/src/pyunit/pyunit.cpp`) is unexamined.

Verified: `cmake --build .build -j` green, `xo-build --sweep` unchanged.
`nix-build ci.nix -A xo-websock` NOT run — RC's standing instruction is that
the nix build is too slow for this cadence — so the packaging check for
xo-refcnt's changed public header surface is outstanding.
