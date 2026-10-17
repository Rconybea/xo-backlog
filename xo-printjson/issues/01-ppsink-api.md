# 01 — PrintJson API: take a PpSink instead of a std::ostream

Status: open (design decided 2026-10-16, RC -- see "Decision"; supersedes "Preferred shape: a JsonPpSink")
Type: task
Milestone: ppsink-migration

`xo-printjson` serialises to `std::ostream *` throughout. Change it to write
into a `xo::pp::PpSink`.

**Motivation (RC): if we are printing JSON, the option to prettify it may be
useful.** That is a stronger case than the usual migration argument. JSON has
real nested structure, and indented/wrapped JSON is a thing people actually
want — which is exactly what a PpSink provides and a raw ostream cannot. The
serialiser already knows where every object and array boundary is; it just
throws that structure away as it writes.

Deferred out of the xo-webutil PpSink refactor (2026-08-06), which was scoped to
diagnostic printing only.

## Decision (RC, 2026-10-16)

**`JsonPrintState` holds a `PpSink *`, and its constructor throws unless that
sink's style selects json escaping.  `PrintJson::print_tp()` forwards to the
same interface.**  No `JsonPpSink` type.

Why this rather than the 2026-09-20 `JsonPpSink`: since then printers stopped
taking an ostream -- they take `JsonPrintState &` (`issues/02`, done
2026-10-05), and `PrintJson` builds that state itself.  So `JsonPrintState`
already stands between every printer and the output; it is the natural gate,
and a separate sink type would duplicate it.

```bash
cd "$(xo-sdlc --umbrella2)"
grep -n 'virtual void print_json' -A1 xo-printjson/include/xo/printjson/JsonPrinter.hpp
#   :33  virtual void print_json(TaggedPtr tp, JsonPrintState & state) const = 0;
grep -n 'JsonPrintState(PrintJson' xo-printjson/include/xo/printjson/JsonPrintState.hpp
#   :86  JsonPrintState(PrintJson const * pjson, std::ostream * p_os);
grep -n 'JsonPrintState state' xo-printjson/src/printjson/PrintJson.cpp
#   print_tp builds it
```

The shape:

- **`JsonPrintState(PrintJson const *, PpSink *)`** throws unless
  `sink->style().escape == escapestyle::json` (`.xo-backlog/xo-printjson/issues/04`).
  The message says what was wrong.  `PpStyle` is const once a sink exists,
  so the state cannot fix a sink, only refuse it.
- **Colour is NOT checked (RC).**  Coloured json has a modest use: json for
  a human to read.  A coloured sink's output is not parseable json -- that is
  the caller's choice to make, not printjson's to forbid.  `PpStyle::json()`
  stays colourless, so the default is parseable; a reader-oriented sink is
  `PpStyle::json()` with `color_enabled` turned back on.  (Supersedes the
  "colour" row of the constraint table under "Preferred shape" below.)
- **`print_tp(TaggedPtr, PpSink *)`** builds the state from the sink.  The
  `std::ostream *` entry points (`print`, `print_tp`, `print_obj`,
  `PrintJson.hpp:72,80,85,101`) stay, as thin wrappers: each builds
  `FlatSink(PpStyle::json(), os.rdbuf())` on the stack and forwards.  No
  ostream caller changes -- websock, `EventStore::http_snapshot`
  (`xo-reactor/include/xo/reactor/EventStore.hpp:67`), the python bindings,
  the tests.  `FlatSink` does not buffer -- it `sputn`s straight through
  (`xo-ppsink/src/ppsink/FlatSink.cpp:55,84`) -- so the wrapper has nothing
  to flush.  The throw is safe there: the state is built after
  `print_tp`'s `EntryGuard`, which unwinds.
- **`p_os()` stays during the migration (RC)**, backed by a small streambuf
  whose writes call `sink->put(..)`.  Everything then reaches the sink, in
  order -- which matters for a `PrettySink`: it buffers tokens to decide
  layout, so a write that bypassed the sink to the underlying streambuf
  would come out reordered.  `json_quot` keeps working (it writes
  already-escaped bytes through `os.rdbuf()`, i.e. the adapter).  Printers
  then move to the sink one at a time; the adapter and `p_os()` go at the
  end.  Nothing in the tree provides this adapter today:
  `PpSink::stream_open()` (`xo-ppsink/include/xo/ppsink/PpSink.hpp:221`)
  hands out a streambuf per TOKEN, not a long-lived one.

  This answers question 1 below ("replace, or add?"): add, then retire,
  one printer at a time.
- **`print_tp` does not call `complete()`** (it writes `"\n"`,
  `PpSink.hpp:201`): the sink is the caller's, and so is finishing it.  With
  a `PrettySink`, output emerges when the outermost group closes -- once
  printers emit `begin`/`end` around objects, the closing `end()` flushes.

Questions 2 and 3 are answered too (2026-10-16), under "Questions to settle
first" below.

### Scaffolding -- built to unwind (RC, 2026-10-16)

Step 1 (`JsonPrintState` takes a `PpSink *`) adds pieces that exist only to
make the migration incremental.  Each goes once nothing needs it:

| piece | goes when |
|---|---|
| `detail::SinkStreambuf` and `JsonPrintState::p_os()` | the last printer writes through `state.sink()` |
| `PrintJson::print_tp(TaggedPtr, std::ostream *)`, and the `print` / `print_obj` ostream overloads that reach it | no caller passes an ostream |
| `json_quot` (`include/xo/printjson/json_quoted.hpp`) | its `os << ..` sites are sink calls (`quot()` through a json sink) |
| tests building `JsonPrintState` from `FlatSink(PpStyle::json(), ss.rdbuf())` over a `std::stringstream` | see below |

**Tests follow the pattern production code should (RC).**  Not for the
tests' own sake: a test is also an example, so it should get its scratch
memory the way regular code would -- from the scratch facilities
(`TempArena`; `tostr()` / `tojsonstr()` build on it,
`xo-indentlog2/include/xo/indentlog2/print/tostr.hpp`), not a heap
`std::stringstream`.  Today there is no facility for "a json sink over
scratch memory whose text I read afterwards" -- `tostr`'s body is one, but
not exposed as a sink.  Proposed (not built): a RAII scratch sink beside
`tostr()`, owning `ArenaReset` + `LogBufferAdapter` + `LogStreambuf` +
`FlatSink(style)`, exposing `sink()` and `output()`; `tostr_styled` becomes
its user.  Nesting on `TempArena` is safe when LIFO -- `LogBufferAdapter`
takes all available space, an inner `tostr` lands beyond it and is reset
before the outer writes again (`xo-indentlog2/src/indentlog2/LogBuffer.cpp`,
`char_expand_to`) -- which a json print is: `map_key_string`'s `tojsonstr`
runs and finishes inside it.

**Corrected 2026-10-16: that nesting claim is WRONG** (kept above per
CONVENTIONS rule 6).  It was reasoned from the code, not run.  Running it
(`TempFlatSink`'s first tests): a `tostr()`, or a second scratch sink, made
while one is alive writes NOTHING -- `""`.  Cause:
`DArena::expand(target_z)` takes an ABSOLUTE size from the arena's start
(`xo-arena/src/arena/DArena.cpp:665`, `target_z <= committed_z_`), but
`LogBufferAdapter::char_expand_to` passes its buffer's own size
(`xo-indentlog2/src/indentlog2/LogBuffer.cpp:113-118`).  Right only for a
buffer at offset 0.  An inner buffer starts where the outer one -- which
took all committed space -- ends, so its request reads as already
satisfied, nothing is committed, and it allocates 0 bytes.  This is a
pre-existing defect of `tostr()` itself: any `tostr` nested in another's
rendering gets `""` today.

**Built 2026-10-16** (uncommitted in the umbrella at the time of writing):
`xo::pp::TempFlatSink` (`xo-indentlog2/include/xo/indentlog2/print/TempFlatSink.hpp`),
and the nesting defect fixed -- `LogBufferAdapter::char_expand_to` now
passes `expand()` the buffer's offset plus its size, the offset from
`DArena::allocated_before(buf_ckp_)` (new; RC's shape).  Pinned by
`xo-indentlog2/utest/TempFlatSink.test.cpp` (`...-survives-nested-tostr`,
`...-uses-its-style`), which failed before the fix.  The 19 test sites that
built a `JsonPrintState` now take a `TempFlatSink`; tests that print through
the ostream entry points stay on `std::stringstream` until those go.

Also found here: `.xo-backlog/xo-printjson/issues/10` -- doubles print at 6
significant digits.

### Re-measured 2026-10-16: the API surface below is stale

Printers take `JsonPrintState &`, not `std::ostream *` (above).  There are
**14** `JsonPrinter` subclasses, not 9 -- 11 in `PrintJson.cpp`, 2 in
xo-kalmanfilter's `EigenUtil.cpp`, 1 in xo-stringtable2's
`SetupStringtable2.cpp` -- and 19 `p_os()` reads, which is what the
migration converts:

```bash
cd "$(xo-sdlc --umbrella2)"
grep -rn ': public JsonPrinter' xo-*/src xo-*/include | awk -F: '{print $1}' | sort | uniq -c
grep -rn 'p_os()' xo-*/src xo-*/include | awk -F: '{print $1}' | sort | uniq -c
#   PrintJson.cpp 11, JsonObject.cpp 3, EigenUtil.cpp 2, JsonMembers.cpp 1,
#   JsonPrintState.cpp 1, JsonPrintState.hpp 1  (19)
```

`print_aux` no longer exists.  Kept below as first written.

## Current API surface (as first written -- stale, see above)

`xo-printjson/include/xo/printjson/PrintJson.hpp`:

```cpp
template <typename T>
void print(T const & x_arg, std::ostream * p_os) const;   // :31
void print_tp(TaggedPtr tp, std::ostream * p_os) const;   // :39
void print_obj(rp<SelfTagging> const & obj, std::ostream * p_os) const;  // :44
void print_aux(TaggedPtr tp, std::ostream * p_os) const;  // :55
```

`xo-printjson/include/xo/printjson/JsonPrinter.hpp`:

```cpp
virtual void print_json(TaggedPtr tp, std::ostream * p_os) const = 0;   // :32
void report_internal_type_consistency_error(TypeDescr, TypeDescr,
                                            std::ostream * p_os) const;
```

**`print_json` is a pure virtual with 9 subclasses tree-wide** — that is the
bulk of the change, and it cannot be done incrementally per-subclass without
carrying both signatures for a while.

## Blast radius

Subsystems naming PrintJson, re-measured 2026-08-09 — **xo-websock 5,
xo-kalmanfilter 3, then one file each in xo-reflect, xo-reactor, xo-pywebsock,
xo-pyreactor, xo-pyprintjson, xo-pyprocess** — plus xo-printjson itself, **5
files, not 7**:

```bash
grep -rl 'PrintJson' --include=*.hpp --include=*.cpp xo-*/ \
  | grep -v '/\.build/' | awk -F/ '{print $1}' | sort | uniq -c | sort -rn
```

The 9 `print_json` implementations are confirmed at 9
(`grep -rn ': public JsonPrinter'`): six in `PrintJson.cpp`, two in
xo-kalmanfilter's `EigenUtil.cpp`, and `AsStringJsonPrinter` in
`JsonPrinter.hpp`. NB they declare `virtual void print_json(...)` without the
`override` keyword, so a grep for `override` finds none of them.

**Update 2026-10-15:** `AsStringJsonPrinter` is deleted (unused, and it
wrote `operator<<` output unescaped -- `.xo-backlog/xo-printjson/issues/04`),
so one fewer subclass to convert.  The count above is not re-measured here:
the printers retired since (`xo-websock/issues/16`) moved it too.

```bash
cd "$(xo-sdlc --umbrella2)" && grep -rn ': public JsonPrinter' xo-*/src xo-*/include
```

The awkward consumer is **`xo-reactor/include/xo/reactor/EventStore.hpp:67`**,
where an `HttpEndpointDescr`'s endpoint fn calls `http_snapshot(pjson, p_os)`
with the ostream that carries the HTTP response body:

```cpp
auto http_fn = ([this, pjson_rp](std::string const &, Alist const &,
                                 std::ostream * p_os)
                { this->http_snapshot(pjson_rp, p_os); });
```

`HttpEndpointFn`'s `std::ostream *` is **deliberately staying** (decided
2026-08-06): it carries HTTP payload, not diagnostics, and an endpoint may
produce anything, not just JSON. So this ticket has to bridge the two — a
`FlatSink` over the response ostream at the http_snapshot boundary is the
obvious answer, and it is also where a future "pretty-print this JSON response"
switch would live.

## Re-measured 2026-09-20

**Nine `print_json` implementations is now thirteen**, and two of the four new
ones are OUTSIDE xo-printjson:

```bash
grep -rn ": public JsonPrinter" --include=*.hpp --include=*.cpp xo-*/ | grep -v "\.build"
```

| | |
|---|---|
| `xo-printjson/src/printjson/PrintJson.cpp` | 8 (was 6): + `JsonPrinter_flatstring`, `JsonPrinter_address`, `JsonPrinter_ObjectSlot` |
| `xo-kalmanfilter/src/kalmanfilter/EigenUtil.cpp` | 2 |
| `xo-printjson/include/xo/printjson/JsonPrinter.hpp` | `AsStringJsonPrinter` |
| `xo-stringtable2/src/stringtable2/SetupStringtable2.cpp` | `DStringJsonPrinter` -- **new subsystem on the list** |

That sharpens question 1. With implementations in three subsystems, a hard
swap of the pure virtual breaks xo-kalmanfilter and xo-stringtable2 in the same
commit. A bridging default on the base class is no longer just tidier; it is
what makes the change land in more than one step.

## The PpSink generalization this needs -- and a ticket it collides with

Anticipated by RC 2026-09-20: "may require some modest generalization of the
PpSink api". Measured, there are two halves, and they pull in opposite
directions.

**Structure: PpSink already has the vocabulary.** `begin()` / `split()` /
`end()` / `newline()` (`xo-ppsink/include/xo/ppsink/PpSink.hpp:164-193`) are
exactly the token stream a nested renderer emits, and `pretty_struct.hpp` is a
worked example of using them -- it emits `begin(0)`, a `put(":")` per field,
`split(1, offset)`, `end()`, and lets `PpState` decide the layout. A JSON
object or array is the same shape. Nothing obvious is missing here.

**Escaping: PpSink's vocabulary is the WRONG one, and adopting it would make
that permanent.** `put_with_escape(sv, quote_flag)` escapes for *display* via
`Escape`, not for JSON. That is already a live defect --
`.xo-backlog/xo-printjson/issues/04`: an embedded NUL renders `\x00` where
JSON requires `\u0000`, so a strict parser rejects the document.

Today printjson escapes on its own account (`quot()` in `JsonPrinter_string`),
so 04 is fixable inside xo-printjson. **If serialisation moves to
`put_with_escape`, the wrong escape vocabulary moves into the sink layer**,
where the fix would then have to be a JSON mode on PpSink rather than a change
to one printer.

So 04 is not a blocker in the usual sense -- this ticket could land first and
make 04 harder. Either settle 04 first, or decide here that printjson keeps
owning its escaping and writes pre-escaped text through plain `put()`. The
second is probably right: escaping is a property of the FORMAT, and PpSink is
format-agnostic.

Worth stating because it inverts the usual reading: moving to a richer sink
normally absorbs responsibilities. Here one of them must be deliberately kept
out.

### A third option: an escape POLICY on the sink (RC, 2026-09-20)

Better than either of the above, and it has a home already. `PpSink` carries a
`PpStyle` -- "the presentation choices in force for this sink"
(`xo-ppsink/include/xo/ppsink/PpSink.hpp:229`) -- and an escape vocabulary is
exactly such a choice. A json-configured sink escapes for json; a diagnostic
sink escapes for display; `put_with_escape` stays one call.

The ownership story is good and needs no new machinery: `style_` is copied
from process-wide defaults AT CONSTRUCTION and exposed only as `const style()`
(`PpSink.hpp:243-248`). So the policy is per-sink and fixed once built --
printjson never mutates a caller's sink, and whoever constructs a json sink
chooses json escaping.

**Which means printjson should VALIDATE rather than impose.** `print(x, sink)`
cannot set the policy on a sink handed to it, so it should check and refuse --
the same shape as `DHandleStore` checking its storage arena's base alignment,
and for the same reason: the invariant is real, unverifiable after the fact,
and cheap to assert at the boundary.

**The one real cost.** `Escape` is "a class used as a namespace: no instances,
all members static" (`xo-ppsink/include/xo/ppsink/escape.hpp:26-40`), entirely
`constexpr` -- a COMPILE-time policy. Making it runtime-selectable is not only
dispatch:

```
Escape::c_max_char_expand = 4      // \x1b   -- display escaping
json needs                  6      // \u0000
```

That constant feeds buffer pre-sizing (`str_size()` -> `stream_open(min_z)`).
Json escaping expands MORE than display escaping, so any sizing that trusts 4
under-allocates under a json policy. The sizing path has to become
policy-aware, not just the emitting path.

So the generalization is slightly less modest than "add an enum". It is still
the right shape -- it puts the choice where the other presentation choices
already live, and it is the only one of the three options that leaves
`put_with_escape` usable by printjson at all.

**Update 2026-10-15:** `.xo-backlog/xo-printjson/issues/04` now builds
this policy -- the escape vocabulary on `PpStyle`, a policy-aware `Escape`
(sizing included), and a `PpStyle::json()` preset -- so 04 lands first and
makes this ticket easier, not harder.  `JsonPpSink` below stays here.

### Preferred shape: a JsonPpSink, and JsonPrinter takes only that (RC, 2026-09-20 -- SUPERSEDED 2026-10-16)

*Superseded by "Decision" above: `JsonPrintState` is the gate, with a runtime
check, not a separate sink type.  Kept per CONVENTIONS rule 6.  Its constraint
table still stands for escaping and `complete()`; its colour row is revised --
colour is the caller's choice (RC, 2026-10-16).*


Rather than printjson validating a sink it was handed, make the constraint a
TYPE: `JsonPpSink` enforces at construction, and `JsonPrinter::print_json`
takes `JsonPpSink &`. Then there is no runtime check, no failure mode to
report, and no way to reach the serialiser with a sink that would corrupt the
document.

Strictly better than the DHandleStore parallel drawn above. DHandleStore
checks at runtime because a `DArena` cannot be a distinct type per alignment;
a sink CAN be.

**The constraints are plural, and escaping is not the dangerous one.**

| | |
|---|---|
| escape vocabulary | json, not display -- `\u0000` not `\x00` (issues/04) |
| **colour** | `PpStyle` carries ANSI colour specs and a styled sink emits them INTO the stream. `xo-ppsink/utest/PpStyle.test.cpp:89` pins it: `"\033[31m:k\033[0m 1"`. Json written to a coloured sink carries `\033[31m` inside the document -- and ESC is a C0 control json also requires escaped, so it is broken twice |
| `complete()` | writes `"\n"` (`PpSink.hpp:201`); a json document probably should not |

Colour is the one that would have been missed. It manifests only when someone
happens to pass a styled sink, and the corruption is invisible until a parser
sees it.

**One consequence to accept.** `PpStyle` is copied at construction and exposed
only as `const`, so a WRAPPER around an existing sink cannot repair a bad
policy -- it can only reject it. That points at `JsonPpSink` being a concrete
sink (or a factory that builds one), not an adapter over whatever the caller
already holds. Which means "render json into this existing PrettySink" stops
being expressible.

That is probably the right trade -- it is the same thing as saying a json
document is not a diagnostic rendering -- but it should be a decision, not a
discovery. The place it will be felt is `xo-reactor/include/xo/reactor/EventStore.hpp:67`,
where an http endpoint hands over the response ostream: that seam now
constructs a JsonPpSink over the response rather than adapting a sink it was
given.

**One smaller thing to decide:** `PpSink::complete()` writes `"\n"`
(`PpSink.hpp:201`). A JSON document should probably not, and the sink the
caller supplies may be shared -- so whoever finishes a document has to not call
it, the way `DArenaVector`'s pretty bindings already avoid it
("calling .complete() would be broken for sinks that do not forward to a
streambuf").

## Questions to settle first

1. **Replace, or add?** *(Answered 2026-10-16: add, then retire one printer at a time, with `p_os()` kept on an adapter meanwhile -- see "Decision".)* `print(x, PpSink&)` alongside the ostream overloads,
   or a hard swap? With 9 pure-virtual subclasses, "add" means every subclass
   implements both, or a default that bridges one to the other. A hard swap
   is more churn at once but leaves one code path.
2. **Does prettified JSON actually get built in this ticket**, or does it just
   move the plumbing to PpSink so that prettifying becomes possible later? The
   motivation is the former; the safe increment is the latter. Note that
   without emitting `begin`/`split`/`end` around objects and arrays, moving to
   PpSink buys nothing observable — a FlatSink renders identically to the
   ostream it wraps. **The structure emission is the actual feature.**

   *Answered 2026-10-16 (RC): this ticket builds prettified json, but
   incrementally comes first -- so it is fine for pretty output to appear
   only at the end, once every printer writes through the sink and emits
   `begin`/`split`/`end`.  Each step before that keeps flat output
   byte-identical.*
3. **What about `jsonp_impl` / `operator<<`** (`PrintJson.hpp:100,115`)? Under
   the pattern used elsewhere these belong in a `printjson_ostream.hpp`, as the
   opt-in paved road for external users.

   *Answered 2026-10-16: there was nothing to rehome.  `jsonp` / `jsonp_impl`
   / its `operator<<` sat in legacy `namespace xo::print` inside
   `#ifdef NOT_USING` -- never compiled, already so at the 2026-06-06
   subrepo import, no users anywhere in the tree:*

   ```bash
   cd "$(xo-sdlc --umbrella2)"
   git log --format='%h %ad %s' --date=short -S'NOT_USING' -- xo-printjson/include/xo/printjson/PrintJson.hpp
   grep -rn 'jsonp' xo-*/src xo-*/include xo-*/utest xo-*/example   # nothing, after the delete
   ```

   *Deleted (RC).  RC: `operator<<` is wanted only for legacy support, and
   there is no legacy here.  If inline json is wanted later -- json inside a
   log line, `xtag("snapshot", jsonp(x, pjson))` -- the shape is a value
   wrapper with a `Prettifier`, not an `operator<<`.  Under "Decision" a
   diagnostic sink (display escaping) is refused by `JsonPrintState`, so that
   Prettifier renders through a temporary json-styled `FlatSink` (as
   `tojsonstr` does) and `put()`s the result.  Not built: no user.*

## Suggested sequencing

`print_json`'s 9 implementations are the risk. Consider doing the leaf
printers first behind a bridging default, so each conversion is verifiable
before the base class flips.

**Files:**
- `xo-printjson/include/xo/printjson/PrintJson.hpp` — `:31,39,44,55,100,115`
- `xo-printjson/include/xo/printjson/JsonPrinter.hpp` — `:32` and 9 subclasses
- `xo-printjson/src/printjson/PrintJson.cpp`
- `xo-reactor/include/xo/reactor/EventStore.hpp:67` — the ostream/PpSink seam
- `xo-printjson/utest/PrintJson.test.cpp` — asserts against `std::stringstream`

**Done when:**
- JSON serialisation writes into a `PpSink`
- flat output is byte-identical to today (pin this before adding structure)
- a prettified rendering is available, if question 2 says so

## Notes

### Corrected 2026-08-09: the iso8601 blocker does not exist

This ticket previously read:

> Also blocked-adjacent: `xo-printjson` still uses legacy `iso8601` as an
> ostream value (`PrintJson.cpp:384`), which has no ppsink equivalent of that
> shape. [...] Closing that gap is a prerequisite for xo-printjson's own
> indentlog migration, and this ticket would subsume it.

Wrong on all three counts. Measured 2026-08-09:

```bash
# the ppsink equivalent exists, and is exactly the value-wrapper shape claimed
# to be missing
grep -n 'class iso8601\|Prettifier<iso8601>\|put_iso8601' \
     xo-ppsink/include/xo/ppsink/pp_time.hpp
#   :37  void put_iso8601(PpSink &, xo::time::utc_nanos);
#   :54  class iso8601 { ... };
#   :65  struct Prettifier<iso8601> { ... };

# and xo-printjson ALREADY uses it -- xo::pp::, not legacy
grep -n 'using xo::pp::iso8601' xo-printjson/src/printjson/PrintJson.cpp   # :29

# the indentlog migration it was said to block is already done
find xo-printjson/.build -name '*.o.d' | xargs grep -lE 'xo/indentlog/' | wc -l   # 0
```

`PrintJson.cpp:390` reaches it through `pp_time_ostream.hpp`
(`os << iso8601(..)`), which is the correct adapter for code that currently
holds an ostream — not a legacy holdover. When this ticket converts that call
site to a PpSink, `Prettifier<iso8601>` is already waiting; the adapter simply
stops being needed.

So **this ticket has no dependency on `xo-ppsink/issues/02`.** Its one remaining
facility, `print/cond.hpp`, has nothing to do with printjson.

**This was not a misreading — the claim went stale.** Established by dates
rather than guessed at:

| | |
|---|---|
| ticket drafted, per the deferral it cites | 2026-08-06 |
| `eca10aa4 xo-ppsink: + time prettifiers`, which is where printjson switched to `xo::pp::iso8601` | 2026-08-07 |
| ticket committed to the backlog (`1a2905d`) | 2026-08-08 |

```bash
git log --oneline -S'using xo::pp::iso8601' -- xo-printjson/src/printjson/PrintJson.cpp
```

So the claim was true when written, was falsified by work that landed the next
day, and was committed unchanged the day after that. An earlier reading of this
correction blamed a namespace confusion between `xo::pp::iso8601` and legacy
`xo::print::iso8601`; that was invented, and the dates do not support it.

**The general problem: cross-ticket blocker claims are hand-maintained and rot
exactly like the hand-kept milestone lists this backlog already stopped
keeping.** Nothing failed when `pp_time.hpp` gained `Prettifier<iso8601>` and
this ticket kept asserting the gap — the same silent drift as a ticket that
forgets its `Milestone:` line, but without the query that would expose it.

Cheap remedy, until something better exists: **when a facility gap closes, grep
the backlog for tickets naming that facility** before closing the item.

```bash
grep -rln 'iso8601' .xo-backlog/          # would have found this ticket
```

Measure flat output before and after. Two separate incidents on this migration
came from predicting rendering rather than observing it.
