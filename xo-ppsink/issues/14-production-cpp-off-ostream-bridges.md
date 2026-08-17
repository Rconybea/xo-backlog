# 14 — no production `.cpp` includes an `*_ostream.hpp` bridge

Status: open
Type: refactor
Milestone: ostream-containment
Progress: grep -rl '_ostream\.hpp' --include=*.cpp xo-*/ 2>/dev/null | grep -v '/\.build/' | grep -v utest | wc -l

RC, 2026-08-16: *"we can have `_ostream.hpp` headers forever, and many unit
tests will need to include them. But something like `rg -l _ostream.hpp | grep -v
utest | grep -v '\.hpp'` should go to zero. That's not sufficient, but it's
better, and better is good."*

The bridges are the paved road **out** of xo — for tests, REPLs, and newcomers
holding a `std::ostream`. Production xo code renders into the `PpSink` it
already has. This counter says whether that separation is real.

It is deliberately weaker than the milestone's end state: a `.cpp` can name
`std::ostream` without including any bridge, so zero here does not mean zero
ostream. `issues/15` is the superset. This one is worth having anyway because it
is cheap, it never false-positives, and it is a strict subset of work that has
to happen regardless.

## Do not read this as a residue of the `display()` conversion

The tempting assumption — *these will fall out as `display(std::ostream&)`
disappears* — is false for most of them, and the split is worth re-deriving
before planning:

```bash
# how many of the offenders also carry a display(), and so fall out for free?
for f in $(grep -rl '_ostream\.hpp' --include=*.cpp xo-*/ | grep -v '/\.build/' | grep -v utest); do
  if grep -q 'display *( *std::ostream\|::display' "$f"; then echo COUPLED
  elif grep -qE 'os *<<|std::cout *<<|std::cerr *<<|ss *<<' "$f"; then echo INDEPENDENT
  else echo "PROBABLY-VESTIGIAL $f"; fi
done | sort | uniq -c | sort -rn
```

Measured 2026-08-16 the coupled share was well under half, so eliminating
`display(std::ostream&)` entirely leaves most of this ticket standing. Three
populations, in increasing cost:

**1. Probably vestigial** — includes a bridge and never streams anything. Same
shape as `xo-reactor/include/xo/reactor/Sink.hpp`, where dropping the include
changed nothing across the whole sweep. These are near-free, and the experiment
is only valid *because* `45fd03bc` made ppsink's fallback opt-in per TU: before
that, an unnecessary bridge include was indistinguishable from a necessary one.
Drop, sweep, keep or restore.

**2. Coupled to a `display(std::ostream&)`** — resolved by that conversion,
which RC has committed to taking to zero. No separate work; just don't count it
twice.

**3. Independent** — streams for some other reason. This is the real conversion
work, and per the milestone's own warning it is *"conversion work wearing an
include-hygiene hat"*: a site doing `os << xtag(..)` usually has no structured
printer at all, so moving the include means writing one.

## `tag_ostream.hpp` dominates, and that is not a distortion

Nearly all the include-sites are `<xo/ppsink/tag_ostream.hpp>`. It is easy to
read that as noise from a low-level utility header and exclude it. Don't: it
supplies `operator<<(std::ostream&, tag_impl<..>)`, so a production file
including it is a production file rendering through an ostream. That is exactly
what this milestone is about, and it is where the bulk of the work is.

```bash
grep -rho '#include [<"][^>"]*_ostream\.hpp' --include=*.cpp xo-*/ \
  | grep -v '/\.build/' | sed 's|.*/||;s|[>"]||' | sort | uniq -c | sort -rn
```

The handful that are *not* xo-ppsink's own leaf formatters are type bridges —
`alloc_ostream.hpp`, `Refcounted_ostream.hpp`, `webutil_ostream.hpp`,
`TypeBlueprint_ostream.hpp`, `type_unifier_ostream.hpp` — and are the sharpest
cases, since a production `.cpp` reaching for one is choosing a flat render of a
structured type.

## Scope note: examples are not tests

`grep -v utest` leaves `*/examples/*` in the count, which is deliberate but
arguable — example code is closer to the newcomer audience the bridges exist
for. If they are exempted, exempt them in the `Progress:` line so the counter
can still reach 0, and say so here rather than in a commit message.

## A stronger successor, once this reaches zero

RC: *"eventually we can also suppress non-`_ostream.hpp` including
`_ostream.hpp`"* — i.e. a bridge may only be included by another bridge, or by a
test. That closes the header side, which this ticket's `--include=*.cpp` filter
deliberately ignores. Worth filing then, not now; the milestone's existing
header sweep already covers part of it.

## Done when

- `Progress:` returns 0
- the vestigial population was resolved by dropping includes, not by converting
  code that did not need converting
- `xo-build --sweep` reads
  `62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`
