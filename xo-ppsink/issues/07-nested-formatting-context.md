# 07 — nested formatting context (significant figures, and whatever else)

Status: open
Type: task

**Deliberately not a proposal.** The deliverable of this ticket is a specific
design; right now there is only a problem statement and some constraints.

Raised by RC 2026-08-09, from the `DFloat` conversion in
`.xo-backlog/xo-printable2/issues/01`.

## The observation

`DFloat` renders `1.0/3.0` as `0.333333`. Measured 2026-08-09, through both the
deprecated and the new protocol — they agree:

| value | rendering |
|---|---|
| `1.0/3.0` | `0.333333` |
| `1e20` | `1e+20` |
| `0.0` | `0` |

Six significant digits. **That is not a decision anyone made** — it is
`std::ostream`'s default, arrived at by omission:

```bash
grep -rn 'Prettifier<double>\|Prettifier<float>' xo-*/ --include=*.hpp | grep -v '/\.build/'
#   (none)
```

There is no `Prettifier<double>`, so a double falls through Prettifier's empty
primary template to `operator<<`. Contrast `Prettifier<int>`
(`xo-ppsink/include/xo/ppsink/Prettifier.hpp:44`), which renders via
`std::to_chars` and is deliberately ostream-free.

So: integers are handled on purpose, doubles are handled by accident, and the
accident is load-bearing — every float in every rendering in xo goes through
it.

## What is wanted

Formatting that **nests**. An enclosing structure should be able to say "render
floats in here to 2 decimal places" and have nested values inherit it, with
inner scopes able to override and restore on exit.

Motivating cases, none of them currently expressible:

- significant figures / fixed vs scientific, per subtree
- a price field at 2dp inside a report otherwise at full precision
- integer base (hex for addresses, decimal elsewhere)
- date/time format for a subtree, rather than per value

## The tension to resolve

xo already has a mechanism for controlling rendering: **value wrappers**.
`quot(s)`, `pad(x, n)`, `iso8601(t)`, `hex_view(p, q)` — each says "render THIS
value THAT way", explicitly and locally.

A nested context is the opposite trade:

| | wrapper | context |
|---|---|---|
| where stated | at the value | at an enclosing scope |
| affects | one value | everything below, until overridden |
| a printer must | know to use it | know nothing |
| surprise | none | a value renders differently depending on where it appears |

The second row is the point — a printer like `DFloat::pretty` should not have
to know about precision — and the fourth is the cost. **A design that makes
`sink.pp(x)` produce different text depending on ambient state is a real
hazard**, particularly for anything that must round-trip (see
`.xo-backlog/xo-object/issues/01`, where Schematika round-tripping is live).

Resolving that tension is what this ticket is for. Wrappers-only, context-only,
or some layering where context sets defaults and wrappers override, are all
plausible and none has been thought through.

## Constraints worth carrying into the design

- **Where does it live?** `PpConfig` holds margins, but those are per-render
  and global to it. Nesting implies a stack pushed/popped around `begin()` /
  `end()`, or an explicit scope object. `PpConfig` is likely the wrong shape.
- **Three tickets now circle the same gap.** This one,
  `.xo-backlog/xo-indentlog2/issues/02` (PrettyContext: configuration as a
  value), and `.xo-backlog/xo-indentlog2/issues/03` (bounded rendering:
  max-items/max-depth, also inherently scoped). They should be designed
  together or at least sequenced deliberately — three separate mechanisms for
  "richer sink configuration" would be worse than one.
- **Levelization.** `Prettifier<double>` belongs in xo-ppsink, which sits below
  xo-indentlog2 and knows nothing of PrettySink. Any context mechanism must be
  expressible at the PpSink level, not only the PrettySink level.
- **Cost.** `Prettifier<int>` uses `to_chars` specifically to stay ostream-free.
  A double formatter should not reintroduce `<ostream>` into the hot path;
  `std::to_chars` handles floating point in C++17 onward.

## The wrapper precedent is not hypothetical: legacy `fixed`

`xo-indentlog/print/fixed.hpp` already is the wrapper answer for this exact
case -- `fixed(x, prec)` renders a double at a given precision. Its
implementation is also evidence for the design direction:

```cpp
std::ios::fmtflags orig_flags = s.flags();
std::streamsize    orig_p     = s.precision();
s.flags(std::ios::fixed);
s.precision(fx.prec_);
s << fx.x_;
s.flags(orig_flags);
s.precision(orig_p);
```

Six lines of save/set/restore, purely to stop one value's formatting leaking
into a shared stream. **Legacy did not use ambient stream state; it defended
against it.** So "the sink owns formatting" is not a departure from how xo
worked -- it is what xo was already doing the hard way.

`fixed` has zero real consumers and was missing from the facility inventory
until 2026-08-09; now recorded in
`.xo-backlog/xo-ppsink/issues/02-facility-gaps.md`. Whether ppsink wants a
`fixed` equivalent is part of THIS ticket's decision, since a wrapper and a
context are alternative answers to the same question.

## A prerequisite that is worth doing regardless

**~~Write `Prettifier<double>`~~ — done 2026-08-09.** Independent of any context design, doubles
should not be rendering through an accidental fallback. Doing it alone would:

- make the current behaviour a choice rather than an inheritance
- remove an `operator<<` dependency from a very common path
- require pinning current output first, since changing it changes every float
  rendering in xo — `xo-object2/utest/printable_render.test.cpp` already pins
  six values for exactly this reason

It preserves `0.333333`: `std::to_chars` with `chars_format::general` and
precision 6 is exactly `%.6g`, hence exactly what ostream produced. NB **not**
to_chars' no-precision overload, which gives the shortest round-trip form
(`0.3333333333333333`) and would have changed every float rendering in xo.
Full sweep confirms the tree is unchanged.

Whether 6 significant digits is the RIGHT default is still open, and is now a
one-line change in one place (`c_default_float_precision`) rather than a
property inherited from `<ostream>`.

`Prettifier<double>::print(PpSink & sink, double x)` already receives the sink,
so a nested context can be reached through it without changing the signature --
RC's point, and the reason this was worth doing before the context design
rather than after.

**Files:**
- `xo-ppsink/include/xo/ppsink/Prettifier.hpp` — the empty primary template and
  `Prettifier<int>` at `:44`, the model to follow
- `xo-ppsink/include/xo/ppsink/PpSink.hpp` — where a scoped mechanism would
  have to be expressible
- `xo-object2/src/object2/DFloat.cpp` — the motivating printer
- `xo-object2/utest/printable_render.test.cpp` — current float renderings,
  pinned

**Done when:**
- a specific design is written down here and agreed
- it says explicitly how it relates to value wrappers, and to
  `xo-indentlog2/issues/02` and `03`

## Notes

Do not implement a context mechanism as a side effect of adding
`Prettifier<double>`. They are separable, and the second is a decision about a
default while the first is a decision about an architecture.
