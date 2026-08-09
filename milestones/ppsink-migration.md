# ppsink-migration — retire legacy xo-indentlog

Status: open
Type: milestone

Move every xo subsystem from legacy `xo-indentlog` to `xo-ppsink` (+
`xo-indentlog2` where a concrete line-breaking sink is needed), then delete the
legacy subsystem.

## Why

Stated by the user at the outset:

> Prefer xo-indentlog2 because the pretty-printing algo is more performant,
> better levelled, less intrusive on code supplying printers, avoids ostream.
> Code that provides printers need only depend on xo-ppsink. So better
> all-around. However the migration is substantial, because of many printers
> originally written for xo-indentlog that have to migrate. So attacking
> incrementally.

## Where it stands (2026-08-08)

Sixteen subsystems migrated. **Six remain, and they are all one cluster:**

    xo-printable2 -> xo-object2, xo-stringtable2 -> xo-gc
                  -> xo-expression2, xo-tokenizer2

They are gated on a single design question, which is why they did not fall to
the incremental approach that cleared the rest:

**`APrintable::pretty(Copaque data, const ppindentinfo &) -> bool` must become
`pretty(Copaque data, PpSink &) -> void`** (`xo-printable2/include/xo/printable2/detail/APrintable.hpp:51`).

That is ~233 files inside the cluster's blast radius
(`xo-deps --users-of=xo-printable2 --format=names -q`). Unlike every earlier
step, there is no independent leaf to start from — the whole cluster moves
together, so the shape wants settling before any code.

## The precedent to copy

`.xo-backlog/xo-expression/issues/01-ppsink-migration-pilot.md` is the same
conversion, already done, on the older expression/reader stack — deliberately
piloted first so this cluster benefits. It records:

- the two protocols differ on exactly one axis (receiver: `this` vs
  `Copaque data`), which is **orthogonal** to the ppindentinfo→PpSink change,
  so the shape transfers
- the return value is vestigial and becomes `void`
- ~2/3 of conversions are mechanical `pretty_struct` substitutions; the
  hand-written two-pass bodies are the real work
- capture a rendered baseline *before* converting — it is the only way to tell
  a dropped field from a layout convention
- expect to break consumers that were freeloading on the propagated dependency

The toolkit it produced is in place and proven: `pretty_struct`, `struct_open`
(runtime arity, `force_break`), `concat`, `Refcounted_pp.hpp` for handle types.

## Done when

- no subsystem declares an `xo-indentlog` dependency
- `xo-indentlog` is deleted, or explicitly kept with a recorded reason
- `xo-ppsink/issues/02` has no unported facilities left

Progress is a query — `xo-sdlc --milestone=ppsink-migration`. Closing this file
is a deliberate act, not an automatic consequence of the last ticket flipping.
