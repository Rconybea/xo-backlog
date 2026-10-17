# 11 -- JsonPpSink: a json writer with semantic entry points

Status: open (implemented 2026-10-17, uncommitted in the umbrella -- awaiting RC review; see Progress)
Type: refactor
Raised: 2026-10-17 (RC), after `.xo-backlog/xo-printjson/issues/01` landed

Introduce `JsonPpSink`: a wrapper around a `PpSink` whose entry points say
what json content is being written -- `open_object()`, `key(..)`,
`number(..)`, `open_list()`, ... -- instead of printers writing text
(`put("[")`, `quot(k)`, `": "`).

## Why (RC, 2026-10-17) -- and what it reverses

`issues/01` decided (2026-10-16) that printers write through a plain
`PpSink`, with `JsonPrintState` checking the sink's style -- "No `JsonPpSink`
type".  The idea behind that: printers describe structure through the
`PpSink` api, and the sink maps it according to `PpConfig`.

Seeing the working implementation, RC reverses it: **the `PpSink` api is too
low-level to carry that idea.**  `put()` hands the sink a block of text to
appear verbatim; the sink cannot tell a string from a float from a bracket.
Any json-level policy -- number formatting, per-role colour, layout -- has
nothing to act on, because the meaning is gone before the text arrives.  A
wrapper with labelled entry points keeps it.

(`issues/01`'s 2026-10-16 Decision stays as written there, per CONVENTIONS
rule 6; this ticket supersedes it.)

## What it buys

- **Well-formed by construction.**  The wrapper keeps a stack of open
  objects/lists and inserts separators itself, and can check that keys and
  values alternate.  The hand-kept "first item" flags go:

  ```bash
  cd "$(xo-sdlc --umbrella2)"
  grep -rnE 'bool first|first_ = ' xo-printjson/src xo-printjson/include
  #   JsonMembers.hpp/.cpp (first_), JsonPrintState.cpp (2 map lambdas),
  #   PrintJson.cpp (RootSet free/slots, Flywheel pools)
  ```
- **One home for json policy:**
  - layout -- today's `json_open` / `json_sep` / `json_close` tokens
    (`include/xo/printjson/json_layout.hpp`) move inside it;
  - escaping, and the check that the sink escapes for json (now in
    `JsonPrintState`'s constructor);
  - number formatting -- `.xo-backlog/xo-printjson/issues/10` (6-digit
    doubles, NaN / Infinity) becomes one method, `number(double)`;
  - per-role style for human-readable json: keys, strings, numbers
    coloured differently -- possible once the role is known.
- **Printers stop depending on ppsink.**

## Proposed shape (for RC to correct)

1. **Wrapper, not subclass.**  Holds a `PpSink &`; does not expose `put()`,
   so printers cannot fall back to raw text.
2. **Separators automatic**, from the open-container stack -- no `sep()`
   in the api.
3. **Entry points:**
   - containers: `open_object()` / `close_object()`, `open_list()` /
     `close_list()`;
   - `key(std::string_view)`;
   - values: `string(std::string_view)`, `number(..)` overloaded for the
     integer types and `double` (and `float`), `boolean(bool)`, `null()`.
4. **Keys from non-string values.**  json object keys are strings.  Today
   `JsonPrintState::map_key_string` prints a map key into a captured string
   and re-quotes it if it was not a json string (e.g. a number).  If the
   wrapper knows a key is expected, a value written in key position --
   `number(3)` -- can render as the key `"3"` directly, and the capture
   goes.
5. **An escape hatch, `raw(text)`**, documented as a last resort, for text
   that is not a json value.  Current cases:

   ```bash
   grep -n 'error-json-printer-not-found\|internal-error: type mismatch' xo-printjson/src/printjson/*.cpp
   grep -n '"NaN"\|"Infinity"\|"-Infinity"' xo-printjson/src/printjson/PrintJson.cpp
   ```

   The error texts are not json at all; NaN / Infinity are a json
   extension (`issues/10` decides their fate).  A timestamp
   (`JsonPrinter_utc_nanos` writes `"` iso8601 `"`) is a string value: it
   formats first, then `string(..)`.
6. **Where it lives.**  `JsonPrintState` holds the `JsonPpSink`;
   `JsonPrinter::print_json` reaches it through the state.  The entry points
   (`print`, `print_tp`, `print_obj`) still take a `PpSink *` from the
   caller and wrap it.  `JsonObject::key_open()` (returns `PpSink &` today)
   returns the `JsonPpSink &`.

## Sequencing

Incremental, as `issues/01` was -- flat output byte-identical at every step
(websock golden snapshots, every pinned test):

1. `JsonPpSink`, wrapping the sink `JsonPrintState` already holds; the
   style check moves into it.
2. printjson's own writers -- `JsonPrintState`, `JsonMembers`,
   `JsonObject`, `type_keys` -- onto it; their first-flags go.
3. the 14 `JsonPrinter` subclasses: 11 in `xo-printjson/src/printjson/PrintJson.cpp`,
   2 in `xo-kalmanfilter/src/kalmanfilter/EigenUtil.cpp`, 1 in
   `xo-stringtable2/src/stringtable2/SetupStringtable2.cpp`

   ```bash
   grep -rn ': public JsonPrinter' xo-*/src xo-*/include
   ```
4. printers no longer reach the raw `PpSink`; `json_layout.hpp`'s tokens
   are internal to `JsonPpSink`.

`.xo-backlog/xo-printjson/issues/10` (floating point) should follow this
ticket: its fix is `JsonPpSink::number(double)`.

**Done when:**
- every printer writes through `JsonPpSink`'s entry points; nothing outside
  it writes json text to a `PpSink`
- no printer keeps its own first-item flag
- flat and pretty output unchanged (all existing pins and goldens green)
- `raw()`'s remaining uses are listed here, each with why

## Progress (2026-10-17, implemented while RC was away; uncommitted)

All four sequencing steps done, in the umbrella working tree (base `1705ffc7`).
Steps 2-4 landed together, not one by one: once `JsonPrintState`'s own
writers track keys and values through `JsonPpSink`, a printer still writing
raw text after a key leaves that key's value pending, and the next key
throws.  So every writer converts at once.

- `JsonPpSink` (`include/xo/printjson/JsonPpSink.hpp`, `src/printjson/JsonPpSink.cpp`)
  as proposed: wrapper, automatic separators, `open/close_object`,
  `open/close_list(sep_spaces = 1)`, `key`, `string`, `number` (integers per
  `xo::pp::pp_number_integral`, `double`, `float`), `boolean`, `null`, `raw`.
  Misuse throws `std::logic_error`.  Bottom frame: any number of top-level
  values, unseparated.
- `JsonPrintState` holds it (`json()`); `sink()`, `write()` and
  `map_key_string` are gone -- a map key prints in key position.
  `JsonObject::key_open` returns `JsonPpSink &`.  `type_keys` has `write(json)`
  instead of a `Prettifier`.
- Every printer converted: printjson's 10 writing printers, xo-kalmanfilter's
  2 (via `open_list(0)`, keeping `[1,2]` byte-identical).  Delegating printers
  (`TaggedPtr`, `DFloat`, `DString`) unchanged.
- `json_layout.hpp` is private (`src/printjson/`), used only by `JsonPpSink.cpp`.
- No first-item flag remains outside `JsonPpSink`.

`raw()` uses (done-when: listed, each with why):

| where | text | why |
|---|---|---|
| `JsonPrinter::report_internal_type_consistency_error` (`PrintJson.cpp`) | `<internal-error: type mismatch ...>` | a diagnostic, not json |
| `JsonPrintState::print_node` | `<error-json-printer-not-found ...>`, `<error-null-tp>` | diagnostics, not json |
| `JsonPrinter_floatingpoint` | `NaN`, `Infinity`, `-Infinity` | json extension; `issues/10` decides |

*Update 2026-10-17: the last row is gone -- `JsonPpSink::number` renders
non-finite values itself (`issues/10`).  `raw()` now has only the three
diagnostics.*

Tests: `utest/JsonPpSink.test.cpp` (9 cases, two mutation-checked).
`JsonMembers.test.cpp`'s 15 cases now open an object around their members
list -- the old fragment `, "_members_": [..]` cannot be written outside an
object any more -- and compare through a `members_of()` helper, so each
entry's bytes stay pinned.

Verified: flat output byte-identical (websock golden snapshots, kalmanfilter
pins, all printjson pins), pretty tests unchanged, `ctest` 49/49 gcc and
clang, 21/21 introspect browser tests, `xo-build --sweep` 73/47/26.

Answered (RC, 2026-10-17):
- misuse: **exception** (`std::logic_error`, as built).
- frame stack (and `JsonPrintState`'s cycle-detection map): arena-backed,
  in a separate ticket -- `.xo-backlog/xo-printjson/issues/12`.
- `open_list(sep_spaces)`: **dropped** -- xo-kalmanfilter's lists now print
  `[1, 2]` like every other list; `json_eigen.test.cpp` pins updated.

Open for RC (as first written):
- misuse throws `std::logic_error`; elsewhere printjson aborts on printer
  misuse (`JsonPrintState::abort_misuse`).  Which?
- the frame stack is a `std::vector` -- heap, unbounded in principle (bounded
  in practice by `PrintJson::max_depth`).  Under the fixed-reserve principle,
  a fixed-capacity stack sized from `max_depth` may be preferred.
- `open_list(sep_spaces)` exists only to keep xo-kalmanfilter's `[1,2]`
  byte-identical; dropping it would change that output to `[1, 2]`.
