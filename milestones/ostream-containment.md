# ostream-containment — keep `<ostream>` out of xo headers

Status: open
Type: milestone

Every `operator<<` for an xo type lives in a dedicated `<subsystem>_ostream.hpp`
(or `<facility>_ostream.hpp`) header, and no non-test xo header includes one.
Streaming becomes an **opt-in** at the edge — for newcomers, tests, and REPL
output — rather than something a header drags in on everyone's behalf.

Raised by RC 2026-08-10, from the `DVarRef` conversion
(`.xo-backlog/xo-printable2/issues/01-aprintable-pretty-ppsink.md`) and the
follow-up `.xo-backlog/xo-expression2/issues/01-binding-prettifier.md`.

## Why

Three separate problems turn out to be the same problem.

**1. ppsink's `operator<<` fallback is silent.** A type with an `operator<<` and
no `Prettifier<>` takes `pretty.hpp`'s third dispatch branch — it compiles, and
renders as one opaque token with no break points. Nothing diagnoses it. The
header itself says so: *"prefer a `Prettifier<T>` specialization so a type never
lands here"* (`xo-ppsink/include/xo/ppsink/pretty.hpp:29-32`). Containing the
inserters is how you find the types relying on that branch, because after this
milestone the *only* place an xo `operator<<` can be declared is a file whose
name says what it is.

**2. `<ostream>` propagates.** `xo-expression2/.../Binding.hpp:9` includes
`<iostream>` — which additionally instantiates `std::ios_base::Init` in every
translation unit downstream — and six headers include `Binding.hpp`. That is one
instance of a pattern; see the count below.

**3. Streaming defeats structured printing.** An `operator<<` renders flat. A
header-defined printer that does `os << xtag(..)` has already decided its output
cannot participate in an enclosing structure's line breaking, which is precisely
what the ppsink migration exists to undo.

The pattern is not new — twenty headers already follow it (ten `_ostream.hpp`
plus ten spelled `_iostream.hpp`), and
`xo-webutil/include/xo/webutil/webutil_ostream.hpp` documents the reasoning
(PpSink is the intended path inside xo; ostream is for newcomers and tests). This
milestone generalises what those ten already did.

## Where it stands (measured 2026-08-10)

Twenty conforming headers exist, under two spellings:

```bash
find xo-*/include -name '*_ostream.hpp' -o -name '*_iostream.hpp' \
  | grep -v '/\.build/' | sort
#   _ostream.hpp  (10): xo-alloc/alloc_, xo-webutil/webutil_, and eight in
#                       xo-ppsink (hex, log_level, pad, pp_time, pretty,
#                       quoted_char, quoted, tag)
#   _iostream.hpp (10): xo-unit x6, xo-flatstring x2, xo-ratio, xo-timeutil
```

**125 headers remain**, across 26 subsystems. The sweep count is one command —
put it on the sweep ticket as its `Progress:` line when that ticket is written
(`CONVENTIONS.md` rule 5; `xo-sdlc` runs `Progress:` on **tickets**, not on
milestone files, so it is recorded here as a command rather than a field):

```bash
{ grep -rl 'operator<< *( *std::ostream' --include=*.hpp xo-*/include 2>/dev/null
  grep -rl '_ostream\.hpp[">]'           --include=*.hpp xo-*/include 2>/dev/null
} | grep -v '/\.build/' | grep -vE '_(i)?ostream\.hpp$' | grep -v '^xo-ppsink/' \
  | sort -u | wc -l
```

Do not write the number into a ticket — it decays.

### CORRECTED 2026-08-10, same day: the first query undercounted by 6x

This ticket was written saying **47**, from

```bash
grep -rln 'ostream *& *operator<<' ...      # 19 files: WRONG
```

That pattern requires the return type and the function name on **one line**.
xo's dominant style puts them on separate lines:

```cpp
        inline std::ostream &
        operator<<(std::ostream & s, const typeseq & x) {
```

so it matched 19 headers where the real figure is 119. Found by accident:
`DConstant`'s printer renders `typeseq` values, and
`xo-reflectutil/include/xo/reflectutil/typeseq.hpp:115` declares exactly the
inserter above — a type visibly in scope for this milestone that the query said
was not there.

**Why the wrong version was plausible:** it was checked against a known case
(`Binding.hpp:58`, which happens to be single-line) and produced a list whose
contents all looked right. A grep that returns real hits reads as working. The
fix is to match the **parameter** (`operator<<(std::ostream`) rather than the
return type, since the parameter cannot be split from the function name.

This is `CONVENTIONS.md` rule 3 — *what one command would show this is false?* —
and the answer here was "grep for a type you already know is a violator and see
whether it appears". Cheap, and not run.

Two consequences beyond the number:

- **The `_iostream.hpp` naming variant is not one file, it is ten** (xo-unit ×6,
  xo-flatstring ×2, xo-ratio, xo-timeutil). They already follow this pattern
  under a different spelling. The query now excludes `_(i)?ostream.hpp`, which
  counts them as conforming — but that is a decision this milestone owes an
  argument for, not a grep detail.
- **16 of the 125 are xo-indentlog**, which `ppsink-migration` deletes outright.
  So the standing work is nearer 109, and that subset will fall without anyone
  touching it.

Population **B** is the larger one and is overwhelmingly `tag_ostream.hpp` —
header-defined printers doing `os << xo::pp::xtag(..)`. Those want converting to
`sink.pretty_struct(...)`, which is the same work as the ppsink migration and
should follow the same phase-C discipline (pin the rendering, then change it).

By subsystem the largest are xo-reader2 (19), xo-indentlog (16, self-resolving),
xo-reader (15), xo-ordinaltree (11) and xo-kalmanfilter (10).

Two exemptions are baked into the query and should be argued rather than
assumed:

- **`xo-ppsink` itself** — `scope.hpp`, `tostr.hpp` and `tag_ostream.hpp`
  include `pretty_ostream.hpp` deliberately: they *are* the fallback machinery.
  Excluded by `grep -v '^xo-ppsink/'`.
- **`xo-indentlog`** — 16 hits, and the subsystem is deleted outright by the
  `ppsink-migration` milestone, so they resolve themselves. Left in the count
  rather than special-cased, since they are genuinely remaining work.

Naming is not uniform: ten headers spell it `_iostream.hpp` rather than
`_ostream.hpp` (listed in the correction above). Settle the spelling before
doing the sweep — the query currently treats both as conforming, which is a
choice, not a measurement.

## Relationship to `ppsink-migration`

**Downstream of it, not part of it.** That milestone's "done when" is *no
subsystem declares an `xo-indentlog` dependency*; nothing here affects that.
Conversely much of population B is easiest to do *as* each printer is converted,
so the two overlap in practice even though neither gates the other.

Do not fold this into `ppsink-migration`. That one is nearly done (`xo-sdlc
--milestone=ppsink-migration`) and its remaining work is a single cluster; adding
125 files of unrelated sweep would make its progress meaningless.

## Done when

- every `operator<<` for an xo type is declared in a `*_ostream.hpp` header
- no non-test xo header includes one (excepting xo-ppsink's own machinery)
- the sweep count above returns 0
- the naming variant (`_iostream` vs `_ostream`) is resolved one way or the other
- **`.xo-backlog/xo-reflectutil/issues/01-typeseq-ostream.md` is closed.**
  Called out by name because that one was deliberately left half-done on
  2026-08-10 -- `Prettifier<typeseq>` exists and `typeseq.hpp` dropped
  `<iostream>` for `<ostream>`, but the inserter is still there, blocked on the
  legacy `xo-indentlog` `xtag` sites that `ppsink-migration` removes. A
  retained-for-now comment in a header is easy to stop seeing; a ticket carrying
  this milestone is not.

## Notes

**The criterion is the deliverable, more than the sweep is.** Today there is no
way to find a type silently taking ppsink's fallback: grepping for `operator<<`
over-reports (the type may also have a `Prettifier<>`) and under-reports (an
inherited or templated inserter does not match). Once every inserter lives in a
file named `*_ostream.hpp`, "which types stream?" becomes a directory listing
instead of a guess. That is worth more than the individual conversions.

**Expect this to surface missing `Prettifier<>`s, not just move includes.** A
header that today streams `xtag(..)` usually has no structured printer at all —
moving the include out means writing one. Budget accordingly: population B is
conversion work wearing an include-hygiene hat.
