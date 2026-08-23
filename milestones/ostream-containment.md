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

## Where it stands (re-measured 2026-08-22)

| | ticket | 2026-08-17 | 08-22 am | 08-22 pm | 08-23 | target |
|---|---|---|---|---|---|---|
| **A** `_pp.hpp` sidecar headers | `xo-ppsink/issues/13` | 3 | 1 | 1 | 1 | 0 |
| **B** production `.cpp` including a bridge | `xo-ppsink/issues/14` | 90 | 87 | 73 | 59 | 0 |
| **C** production files naming `std::ostream` | `xo-ppsink/issues/15` | 236 | 224 | 182 | 150 | 0 |

The `08-22 pm` column is after three subsystems were cleared in one session --
xo-arena, xo-reflect (17) and xo-expression (39), the latter two being the first
two of the ten listed under "Order of attack". Two same-day columns because the
morning figures are what the `_iostream.hpp` filter correction produced, before
any of that work; keeping both is what makes the filter change and the real work
separable, per the note below.

**B and C are not comparable across that date.** Both filters were corrected on
2026-08-22 for the `_iostream.hpp` spelling (decision 1 below), and the
correction moved them in opposite directions: C fell 10 by filter (ten
conforming headers had been counted as violations), B rose 13 by filter
(thirteen production `.cpp` files including a bridge had been invisible) while
falling 16 by real work. No file in the tree changed on account of the filter.

## Where it stood (re-measured 2026-08-17)

Three counters now carry this milestone, each on its own ticket so each has its
own progress bar. The commands live in the tickets (`CONVENTIONS.md` rule 5);
the dated numbers live here, because a milestone file is where measurements are
allowed to be written down and go stale visibly.

| | ticket | 2026-08-17 | target |
|---|---|---|---|
| **A** `_pp.hpp` sidecar headers | `xo-ppsink/issues/13` | 3 | 0 |
| **B** production `.cpp` including a `*_ostream.hpp` | `xo-ppsink/issues/14` | 90 files | 0 |
| **C** production files naming `std::ostream` | `xo-ppsink/issues/15` | 236 files / 535 lines | 0 |

C is the invariant; A and B are its tractable decompositions. A was 5 on
2026-08-16 and both hazardous entries have since been retired (`e2b8978c`,
`d19d17c3`).

### The `display`-shaped counter misses the largest population

RC undertook on 2026-08-16 that `display(std::ostream&)` reaching zero is a
necessary victory condition. It is necessary and it is **not** the measure of
this milestone, because it is not the only spelling:

```bash
grep -rhoE '[a-z_]+ *\( *std::ostream' --include=*.hpp --include=*.cpp xo-*/ \
  | grep -v '/\.build/' | sed 's/ *( *std::ostream//' | sort | uniq -c | sort -rn
#   display 101, print 72, welcome 5, report 5, dump 2, + a tail
```

**xo-reader and xo-reader2 declare zero `display()`** — every printer in both is
`print(std::ostream&)` — and they are the two heaviest subsystems in C (81 and
59 mentions). A `display`-only sweep finishes with them untouched.

So this milestone's criterion is *any* member taking `std::ostream&`, which is
what counter C measures without having to enumerate method names at all. The
narrower promise still stands as a sub-goal; it just is not the finish line.

Recorded rather than silently corrected, per rule 6: the `display` reading was
plausible because it is the dominant spelling, it is what every ticket in
`ppsink-migration` converted, and grepping it returns a hundred real hits across
seventeen subsystems. Nothing in that result hints at another seventy-two under
a different name. The falsifying command was *"grep the printer method in a
subsystem you believe is unconverted"* — xo-reader, one line, not run.

### Order of attack: per subsystem, in build order, low to high

The unit of work is a subsystem's printer surface, not a counter — all three
counters plus the printer count move together on one pass. Low-to-high for the
same reason `tostr-arena` sequenced that way: a high subsystem's `pretty_struct`
nests low types, and a low type without a Prettifier renders flat (or, since the
sidecars were retired, fails to compile). High-first means redoing the nesting.

```bash
xo-build --list          # the order
```

Ten subsystems carry the bulk. By build position: xo-reflect (17), xo-reader2
(29), xo-alloc (36), xo-ordinaltree (38), xo-expression (39), xo-tokenizer (41),
xo-reader (42), xo-printjson (48), xo-reactor (50), xo-kalmanfilter (61).

Per-subsystem recipe, since it repeats ten times:

1. pin the current rendering in a test — measure, do not predict layout
   (`.xo-backlog/tostr-arena/spec.md` phase-C discipline; two incidents on this
   migration came from predicting)
2. printer -> `pretty(PpSink&)`, `Prettifier<>` beside the type (`issues/13`)
3. drop the `*_ostream.hpp` includes that become unnecessary (`issues/14`)
4. `xo-build -q --configure --with-utests --with-examples --build --install <sub>`
5. `xo-build --sweep` — `62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`

### Three decisions this milestone owes before the sweep can reach zero

**1. The `_iostream.hpp` spelling. SETTLED 2026-08-22 — both spellings conform.**
RC's decision: the ten `_iostream.hpp` headers stay as they are; no rename. The
milestone's sweep query already treated both as conforming, but counter C's
`Progress:` filter (`issues/15`) matched only `_ostream\.hpp`, so the two
queries disagreed about the same ten files for twelve days and C was the
stricter by accident, not by decision. Both `Progress:` lines now use
`_(i)?ostream\.hpp`.

**2. The carve-outs, now that 100% is the bar.** Earlier tickets ruled families
out of scope on defensible grounds, and a counter cannot reach zero while they
stand. Either they are in scope, or they become written exclusions in the
relevant `Progress:` line:

- `GcStatistics` / `ObjectStatistics` (12 decls) — `xo-alloc/issues/01` excluded
  them as non-virtual and outside the `Object` hierarchy. The sharpest of the
  four, because the exclusion had a real argument behind it.
- xo-expression2 / xo-object2 (2) — v2/facet cluster
- xo-symboltable (2) — dormant, never compiles
- xo-jit `MachPipeline.{new,orig}.cpp` (2) — never compile; deletion, not
  conversion (`.xo-backlog/xo-jit/issues/02`)

**5. xo-indentlog STAYS for now. RECORDED 2026-08-23 (RC).** It has zero
dependents and could be deleted today, but its documentation has not been
carried across to xo-indentlog2 -- 14 files / 372K against xo-indentlog2's 2
files / 12K, including eight reference documents with no v2 counterpart. RC is
keeping the subsystem until they are mined:
`.xo-backlog/xo-indentlog2/issues/08-mine-indentlog-docs.md`.

**Consequence for counter C:** its 20 files are permanent non-work while it
stays -- 19% of the 105 remaining on 2026-08-23. Together with xo-ppsink's own
2 machinery files (decision 3), **22 of 105 are counted-but-not-work**. C cannot
reach 0 until both get exclusions in `issues/15`'s `Progress:` line, or until
xo-indentlog is deleted. Do not read a stalled C as a stalled migration; re-read
this note first.

**4. Protocol payload sinks are IN SCOPE. SETTLED 2026-08-23 — there is no
branch where `std::ostream*` is the right type.** Proposed as a third exemption
class alongside decision 3's two, on the grounds that
`DynamicEndpoint::http_response(uri, std::ostream*)` and
`Webserver::dynamic_http_response(..)` (xo-websock) emit an HTTP body to a
socket rather than rendering an xo value for a reader, so a PpSink's
line-breaking has nothing to decide. RC rejected it, and the reasoning
generalises:

- The payload is HTML or JSON, and **it is read by humans** -- in
  DynamicEndpoint's case it appears verbatim in browser devtools. Line-breaking
  that makes it readable there is wanted, so PpSink is not the wrong instrument
  after all.
- And where line-breaking genuinely does not apply, the answer is still not an
  ostream: it is an arena-backed `std::streambuf *`.

So both branches lead away from `std::ostream*`, which is why this is not an
exemption but deferred work. Consistent with `xo-printjson/issues/01`, which
already makes the same argument for the JSON serialiser: *"JSON has real nested
structure, and indented/wrapped JSON is a thing people actually want."*

**Affected, and PARKED (RC 2026-08-23) rather than exempted:** xo-websock
`DynamicEndpoint.{hpp,cpp}` + `Webserver.cpp` (3 files), and by the same
argument `xo-printjson`'s `print_json(TaggedPtr, std::ostream*)` -- whose
overriders include `xo-kalmanfilter/src/kalmanfilter/EigenUtil.cpp`. Do not
absorb these into a subsystem pass; they are an interface change spanning
xo-printjson and its implementors, which is what `xo-printjson/issues/01`
already tracks.

**3. What a carve-out has to argue. SETTLED 2026-08-22 — ostream in the API,
not ostream in the file.** RC's rule, from clearing xo-arena: a stream mention
disqualifies a file only when a *caller* can see it. Two consequences.

- A `std::cerr <<` inside a function body whose signature names no stream is an
  implementation detail. `xo-arena/src/arena/backtrace.cpp:93` is the model
  case — one `cerr` on the non-Linux branch of a `void`-returning function.
  Tidy it or don't; it is not this milestone.
- A file whose *purpose* is the ostream bridge is exempt by construction.
  `xo-ppsink/FlatSink.hpp`, `xo-ppsink/pretty.hpp`,
  `xo-indentlog2/PrettySink.hpp`, `xo-arena/arena_streambuf.hpp`: the ostream
  dependence is in the name. Requiring these to be ostream-free is requiring
  them not to exist.

This retires the "argued rather than assumed" debt on the `grep -v
'^xo-ppsink/'` exemption above, and generalises it: the exclusion is not "this
subsystem is special", it is "this file's job is the boundary". Note it does
NOT settle decision 2 — those four families all put `std::ostream&` in a
signature, so they are excluded on other grounds (dormant, never-compiles,
outside a hierarchy) and still owe their own argument.

**Known limit, stated so it is not rediscovered.** Counters B and C grep
`std::ostream` and a bridge-header include. Neither sees `std::cerr`,
`std::cout` or `std::streambuf`. Under this criterion that is correct where
those sit in a `.cpp` body — and wrong where one reaches an API, e.g. a
constructor taking `std::streambuf*`. The counters reaching 0 is therefore
necessary, not sufficient; the per-subsystem check is the wider grep:

```bash
grep -rn 'std::ostream\|std::cerr\|std::cout\|std::clog\|std::streambuf' \
     --include=*.hpp --include=*.cpp <sub>/include <sub>/src
```

**First subsystem cleared under this rule: xo-arena (2026-08-22).** Zero rows in
B and C. `AllocError` and `cmpresult` gained `pretty(PpSink&)` + `Prettifier<>`
and `xo/arena/print.hpp` was deleted outright; the surviving `cerr` in
`backtrace.cpp` and the whole of `arena_streambuf.{hpp,cpp}` are exempt under
the rule above — the latter additionally having zero users tree-wide
(`grep -rn arena_streambuf` finds only its own `CMakeLists.txt` line), so it is
a deletion candidate for unrelated reasons.

**Second: xo-reflect (2026-08-22), build position 17.** The heaviest of the ten
subsystems listed under "Order of attack" below, and the first of them done.
B and C both 0; verified `xo-build --sweep` green (`62 attempted: 34 ok, 28
with no tests, 0 failed, 0 skipped`).

What it took, since the recipe below understates two of the steps:

- `pretty(PpSink&)` + `Prettifier<>` for `TypeDescrBase`, `TypeDescr`, `TypeId`,
  `Metatype` (new `metatype2str`, matching `comparison2str` /
  `AllocError::error_description`) and `TaggedRcptr`.
- `print_reflected_types(std::ostream&)` -> `(PpSink&)`, and its three callers
  (`xo-pyreflect/src/pyreflect/pyreflect.cpp`, `xo-ratio` and `xo-process`
  utests) rebuilt around `FlatSink sink(PpStyle::colored(), std::cout.rdbuf())`
  rather than a `pp_to_stream` bridge. Worth copying: the bridge would have put
  `pretty_ostream.hpp` into a production .cpp and moved the violation from
  xo-reflect to xo-pyreflect. A `streambuf` in a function body is exempt under
  decision 3; a bridge include is not.

Two lessons, per rule 6:

- **`#ifdef OBSOLETE` does not reduce either counter.** Guarding the old
  `display()` / `operator<<` left 12 counter-C lines standing in code that does
  not compile, and blocked removing `<iostream>` from `TypeDescr.hpp` (a header
  with 34 consumers, i.e. the reason-2 case in this milestone's Why). Delete;
  do not guard. Same blind spot as the `FacetRegistry::dump()` incident: a text
  grep cannot see preprocessor state.
- **A wrong `Prettifier<T>` is indistinguishable from no `Prettifier<T>`.** A
  specialization supplying `pretty()` instead of `print()` fails
  `has_prettifier` silently, and dispatch falls through to the operator<<
  fallback -- which compiles wherever `pretty_ostream.hpp` happens to be in
  scope. That is thesis 1 of this milestone reproducing itself inside the
  conversion meant to fix it. It becomes a hard error only once the inserter is
  actually deleted, which is the argument for deleting inserters EARLY in a
  subsystem rather than last.

**Output-visible change, recorded because this milestone promised text
preservation:** a null `TypeDescr` now renders `"<nullptr>"`. It previously
rendered nothing through `Prettifier<TypeDescr>` (preserving legacy
`ppdetail<TypeDescr>`) and `"<nullptr>"` through the ostream inserter; retiring
the inserter would have made silence the default by omission rather than by
decision. RC chose `"<nullptr>"`. Every other rendering in xo-reflect passed
through the conversion byte-identical.

**Third: xo-expression (2026-08-22), build position 39.** B and C both 0;
verified `xo-build --sweep` green (`62 attempted: 34 ok, 28 with no tests,
0 failed, 0 skipped`, and the `--sweep ok (build and utest)` line -- read that
line, not the totals, per the note in CONVENTIONS.md).

Two patterns worth copying, neither of which appears in the recipe below.

- **A temporary bridge should be `os << tostr(x)`, not a second printer.**
  `SymbolTable::print(std::ostream&)` was a pure virtual with overrides in
  GlobalSymtab and LocalSymtab, and could not simply be deleted: an
  `rp<SymbolTable>` streamed in xo-reader reaches it through
  `xo-refcnt/Refcounted_ostream.hpp`'s `operator<<(ostream&, intrusive_ptr<T>)`,
  which no grep for `->print(os)` finds.  RC retired the pure virtual anyway and
  segregated the inserter into a new `SymbolTable_ostream.hpp` implemented as
  `os << tostr(x)` -- i.e. routed through the existing Prettifier.  So there is
  ONE implementation, and the drift that made xo-reflect's
  `ostream_baseline.test.cpp` necessary cannot arise.  Prefer this to a
  hand-written `x.print(os)` body in any bridge this milestone creates.

- **`xtag` in a `scope log(..)` needs `tag.hpp`, NOT `tag_ostream.hpp`.**  Nine
  of xo-expression's counter-B hits were dead bridge includes that survived the
  conversion because `xtag` was still visible in the file, so the include looked
  load-bearing.  It is not: `xtag` comes from `tag.hpp`; `tag_ostream.hpp` is
  needed only to stream a tag TO an ostream, which a `scope log(..)` does not
  do.  Expect this in every remaining subsystem -- it is the difference between
  counter B falling and counter B looking stuck after the real work is done.

**A leftover the counters cannot see:** `exprtype.hpp` kept `#include <ostream>`
after its `operator<<` was removed, with nothing in the file using it.  Counter
C matches `std::ostream`, not the include, so this would have sat in a widely
included header indefinitely -- the reason-2 propagation case surviving the
conversion meant to end it.  Worth a grep for orphaned `<ostream>` / `<iostream>`
includes at the end of each subsystem pass.

**Still owed by xo-expression:** `SymbolTable_ostream.hpp` is explicitly
TEMPORARY, and is a counter-B hit in **xo-reader**
(`src/reader/envframestack.cpp`), not here.  Retiring it belongs to xo-reader's
own pass -- now the heaviest remaining subsystem at B=18 / C=36.  The other two
bridges (`type_unifier_ostream.hpp`, `TypeBlueprint_ostream.hpp`) are ordinary
opt-in surface with test-only consumers.

**In flight: xo-reader (2026-08-23), build position 42.** Was the heaviest
remaining subsystem (C=36 / B=18 on 08-22); now **C=4 / B=4**. Not cleared --
what is left is listed below, and three of the eight are the `examples/`
question rather than work.

This pass introduced tree-wide machinery, in `xo-ppsink`:

- **`prettifier_broken<T>` + a `static_assert` at the top of `pretty()`**
  (`Prettifier.hpp`, `pretty.hpp`). Turns "Prettifier<T> specialized but
  unusable" from a silent fallback into a one-line compile error. The assert
  MUST sit before the `if constexpr` chain: the pre-existing terminal
  static_assert is unreachable in this failure, because `pp_streamable` is
  unconstrained and swallows any T once the TU has seen `pretty_ostream.hpp`.
  Verified against the real failure shape.
- **`XO_PRETTIFIER_DECLARE` / `_VIA_PRETTY_METHOD` / `_VIA_CONVERSION`.**
  13 call sites in xo-reader. NOTE the pairing constraint, documented at the
  macro definition: DECLARE in the header, VIA_* in exactly one .cpp. The VIA_
  expansion is an out-of-line member of an explicit specialization and is
  therefore NOT implicitly inline -- putting it in a header gives
  `multiple definition` at link. Marking it `inline` does not fix this, it
  inverts it: an inline function must be defined in every TU that odr-uses it,
  so the definition-in-one-cpp arrangement then fails with
  `undefined reference` instead. Both failure modes were checked; the two
  spellings genuinely cannot share one macro.

A third silent-Prettifier failure mode surfaced here, distinct from the two in
the xo-reflect entry: `template <> class Prettifier<T> { static void print(..) };`
-- a `class` specialization with no `public:`. Access checking is part of a
requires-expression, so `has_prettifier<T>` is false and dispatch falls through
exactly as if no specialization existed. That is what `prettifier_broken` now
catches.

**Still open in xo-reader:**

- `reader_error.hpp:41,44` -- `print(os)` AND `report(os)`, both forwarding to
  `tk_error_`. Gated on **xo-tokenizer** (position 41), not on xo-reader.
  `report()` is user-facing error reporting rather than diagnostics
  (`reader.cpp:101-117`), which is a different contract from `pretty()` and
  wants a deliberate decision rather than a reflex conversion.
- `formal_arg.hpp:34` -- legacy `print(std::ostream&)`, defined inline.
- `examples/exprrepl` and `examples/exprreplxx` -- 2 of C and 1 of B. Blocked on
  the standing `example/` question (see below), not on any conversion.
- 3 `.cpp` bridge includes (`apply_xs`, `envframestack`,
  `expect_formal_arglist_xs`).

**`example/` -- DEFERRED by decision, 2026-08-23 (RC).** Not "unsettled": the
ordering is the answer. *Nothing that can reach across subsystems is finished,
so nothing with a narrower blast radius gets attention yet.* An `example/`
translation unit is a leaf -- no other subsystem includes it, so its `<ostream>`
cannot propagate into anyone else's TUs, and propagation is the harm this
milestone exists to stop (reason 2 in Why). Headers are the opposite extreme and
go first; `src/*.cpp` next; `example/` last.

Recorded because the reviewing pressure ran the other way: `example/` was
flagged as urgent on the grounds that it is a visible share of both counters and
blocks reaching 0. That is a property of the bookkeeping, not of the code. **A
counter is a proxy for blast radius, and where the two disagree, blast radius
wins** -- the same reasoning that makes `arena_streambuf` (zero users) and a
`.cpp`-body `cerr` exempt under decision 3, and the same reason
`xo-indentlog`'s 20 files are not real work. Do not let counter arithmetic
reorder the queue.

Consequence for reading the table above: **A, B and C cannot reach 0 until the
`example/` tail is picked up at the end.** A subsystem showing a small nonzero
count made up entirely of `example/` files is DONE for sequencing purposes.
xo-reader is the first to be in that state (2 of its 4 C, 1 of its 4 B).

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
- **counters A, B and C all return 0** (`xo-ppsink/issues/13`, `14`, `15`).
  C is the one that decides this milestone: it is indifferent to how a mention
  arrived, so it cannot be satisfied by converting only the printer spellings
  someone thought to grep for
- **no member taking `std::ostream &` remains in production code** — `display`
  AND `print` AND the tail, not `display` alone
- the naming variant (`_iostream` vs `_ostream`) is resolved one way or the other
- **`.xo-backlog/xo-ppsink/issues/10-verify-inserters-unused.md` is closed.**
  Containment says where the inserters are; that ticket says whether anything
  still calls them, by compiling the definitions out and seeing what breaks.
  Sequenced immediately after `ppsink-migration`, since that milestone deletes
  xo-indentlog and a large share of the inserters with it. If the live set comes
  back empty, the bridges get DELETED rather than merely contained -- so this
  milestone's real end state is decided there, not here.
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
