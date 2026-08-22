# 15 — no production xo file names `std::ostream`

Status: open
Type: refactor
Milestone: ostream-containment
Progress: grep -rl 'std::ostream' --include=*.hpp --include=*.cpp xo-*/ 2>/dev/null | grep -v '/\.build/' | grep -v '/utest/' | grep -vE '_(i)?ostream\.hpp' | wc -l

RC, 2026-08-16: *"eventually we can also [...] grep for `std::ostream` itself."*

This is the terminal counter for `ostream-containment`. It subsumes `issues/13`
(retire the `_pp.hpp` sidecars) and `issues/14` (production `.cpp` off the
bridges), and it does not care how the mention arrived — a printer signature, a
bridge include, a raw inserter, a `std::ostream &` parameter passed through.
Those two are the tractable decompositions; this one is the invariant.

Counted in **files**, not lines: the file is the unit of work, and a file with
eleven mentions converts once.

## Both bridge spellings count as conforming — decided 2026-08-22

RC settled the naming question the milestone had carried open since 2026-08-10:
`_iostream.hpp` and `_ostream.hpp` are **both** conforming spellings, and the
ten headers using the former are not work.

The `Progress:` filter above said `_ostream\.hpp`, which does not match
`_iostream.hpp` — so all ten were counted as violations:

```
xo-flatstring  int128_iostream.hpp  flatstring_iostream.hpp
xo-ratio       ratio_iostream.hpp
xo-timeutil    timeutil_iostream.hpp
xo-unit        dim_ natural_unit_ quantity_ xquantity_ scaled_unit_ bpu_iostream.hpp
```

The milestone's own sweep query already excluded `_(i)?ostream\.hpp$`, so the
two queries disagreed about the same ten files and this counter was the stricter
of the two by accident, not by decision. Filter corrected to match; the count
drops by 10 without any file changing.

**This lowered C and raised B.** The same spelling gap in `issues/14` ran the
other way — see the matching note there. A filter fix is not a direction.

## Two exclusions, both deliberate

- **`/utest/`** — tests are the audience the bridges exist for. This is the same
  exemption `issues/14` takes and it is the whole reason the bridges survive the
  milestone.
- **`*_ostream.hpp`** — a bridge header naming `std::ostream` is the bridge
  working as intended.

`xo-indentlog` is **not** excluded, following the milestone's own reasoning: its
files are genuinely remaining work, they just happen to resolve when
`ppsink-migration` deletes the subsystem rather than by anyone converting them.
Expect a step change in this counter on that day, and do not read it as progress
on the conversion. As of 2026-08-17 the subsystem still builds
(`find .build -path '*indentlog*' -name '*.o' | grep -v indentlog2`) even though
nothing declares a dependency on it.

## The counter that this one had to replace

The obvious counter — `display(std::ostream&)`, which RC committed to driving
to zero — **misses the largest population in the tree**:

```bash
grep -rhoE '[a-z_]+ *\( *std::ostream' --include=*.hpp --include=*.cpp xo-*/ \
  | grep -v '/\.build/' | sed 's/ *( *std::ostream//' | sort | uniq -c | sort -rn
#   display 101, print 72, welcome 5, report 5, dump 2, + a tail
```

**xo-reader and xo-reader2 declare zero `display()`** and spell every printer
`print(std::ostream&)`. They are the two heaviest subsystems by ostream mentions
— together roughly a third of this ticket. A `display`-shaped counter reads them
as clean.

Plausible and wrong, which is why it is recorded rather than quietly fixed: the
`display` spelling really is the dominant one, it really is what every ticket in
`ppsink-migration` converted, and a grep for it returns a hundred real hits in
seventeen subsystems. Nothing about that result says "and there is another
seventy-two under a different name". `CONVENTIONS.md` rule 3 — the falsifying
command was *"grep for the printer method in a subsystem you already believe is
unconverted and see whether it appears"*, and xo-reader was sitting right there.

The same trap is why this ticket counts `std::ostream` rather than any method
name at all: a spelling census is a lower bound, and this migration has now
learned that four times (include census in `tostr-arena` class B and class E,
the `ostream &` return-type grep in the milestone's own correction, and here).

## Sequencing

Do not work this ticket directly. It closes when `issues/13`, `issues/14` and
the per-subsystem printer conversions have run; the milestone file carries the
build-order sequence and the reasoning for low-to-high. This ticket exists to
say whether they were sufficient — the counter reaching 0 is the only thing that
distinguishes "we converted the printers we knew about" from "no production file
names `std::ostream`".

## Done when

- `Progress:` returns 0
- `xo-build --sweep` reads
  `62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`
- `.xo-backlog/xo-ppsink/issues/10-verify-inserters-unused.md` has decided
  whether the `*_ostream.hpp` bridges are deleted or retained — with this
  counter at 0, nothing in production calls them, which is exactly the input
  that ticket needs
