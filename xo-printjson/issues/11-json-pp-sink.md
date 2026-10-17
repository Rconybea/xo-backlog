# 11 -- JsonPpSink: a json writer with semantic entry points

Status: open
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
