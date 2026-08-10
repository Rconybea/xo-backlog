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

The pattern is not new — ten headers already follow it, and
`xo-webutil/include/xo/webutil/webutil_ostream.hpp` documents the reasoning
(PpSink is the intended path inside xo; ostream is for newcomers and tests). This
milestone generalises what those ten already did.

## Where it stands (measured 2026-08-10)

Ten conforming headers exist:

```bash
find xo-*/include -name '*_ostream.hpp' | grep -v '/\.build/' | sort
#   xo-alloc/alloc_ostream.hpp, xo-webutil/webutil_ostream.hpp, and eight in
#   xo-ppsink (hex, log_level, pad, pp_time, pretty, quoted_char, quoted, tag)
```

**47 headers remain**, across 21 subsystems. The sweep count is one command —
put it on the sweep ticket as its `Progress:` line when that ticket is written
(`CONVENTIONS.md` rule 5; `xo-sdlc` runs `Progress:` on **tickets**, not on
milestone files, so it is recorded here as a command rather than a field):

```bash
{ grep -rl 'ostream *& *operator<<' --include=*.hpp xo-*/include 2>/dev/null
  grep -rl '_ostream\.hpp[">]'      --include=*.hpp xo-*/include 2>/dev/null
} | grep -v '/\.build/' | grep -v '_ostream\.hpp$' | grep -v '^xo-ppsink/' \
  | sort -u | wc -l
```

Do not write the number into a ticket — it decays. It is two populations, and
they need different work:

```bash
# A: declares an operator<< outside an _ostream.hpp  (19 files)
grep -rln 'ostream *& *operator<<' --include=*.hpp xo-*/include \
  | grep -v '/\.build/' | grep -v '_ostream\.hpp$'

# B: a non-test header that INCLUDES an _ostream.hpp, i.e. streams in a header
grep -rn '_ostream\.hpp[">]' --include=*.hpp xo-*/include \
  | grep -v '/\.build/' | grep -v '_ostream\.hpp$' | grep -v '^xo-ppsink/'
```

Population **B** is the larger one and is overwhelmingly `tag_ostream.hpp` —
header-defined printers doing `os << xo::pp::xtag(..)`. Those want converting to
`sink.pretty_struct(...)`, which is the same work as the ppsink migration and
should follow the same phase-C discipline (pin the rendering, then change it).

`xo-ordinaltree` alone accounts for 10 of the 47 and is nearly all population B.

Two exemptions are baked into the query and should be argued rather than
assumed:

- **`xo-ppsink` itself** — `scope.hpp`, `tostr.hpp` and `tag_ostream.hpp`
  include `pretty_ostream.hpp` deliberately: they *are* the fallback machinery.
  Excluded by `grep -v '^xo-ppsink/'`.
- **`xo-indentlog`** — one hit (`print/time.hpp`), and the subsystem is deleted
  by the `ppsink-migration` milestone, so it resolves itself. Left in the count
  rather than special-cased, since it is genuinely remaining work.

Naming is not yet uniform: `xo-unit/include/xo/unit/bpu_iostream.hpp` is
`_iostream`, not `_ostream`, and the query does not match it. Settle the
spelling before doing the sweep, or the criterion has a hole in it.

## Relationship to `ppsink-migration`

**Downstream of it, not part of it.** That milestone's "done when" is *no
subsystem declares an `xo-indentlog` dependency*; nothing here affects that.
Conversely much of population B is easiest to do *as* each printer is converted,
so the two overlap in practice even though neither gates the other.

Do not fold this into `ppsink-migration`. That one is nearly done (`xo-sdlc
--milestone=ppsink-migration`) and its remaining work is a single cluster; adding
47 files of unrelated sweep would make its progress meaningless.

## Done when

- every `operator<<` for an xo type is declared in a `*_ostream.hpp` header
- no non-test xo header includes one (excepting xo-ppsink's own machinery)
- the sweep count above returns 0
- the naming variant (`_iostream` vs `_ostream`) is resolved one way or the other

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
