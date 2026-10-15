# 04 — string rendering escapes with ppsink's vocabulary, not JSON's

Status: open (design revised 2026-10-15 -- see "Design"; supersedes "What the fix is not")
Type: bug

`JsonPrinter_string` renders through `quot()`, which escapes for *display* via
ppsink, not for JSON. The two vocabularies differ, so some output is not valid
JSON.

## What is actually there

Start by deleting a stale claim. `xo-printjson/src/printjson/PrintJson.cpp:367`
reads:

```cpp
/* TODO: escapes special characters */
*p_os << quot(*x);
```

The TODO says escaping is missing. It is not — `quot()` escapes, and has since
the ppsink migration; `xo-ppsink/include/xo/ppsink/quoted_char.hpp` says so in
its header comment, including that it now escapes characters the legacy version
passed through raw. So the bug is not *absent* escaping, it is the *wrong*
escaping, which is a narrower and more findable thing.

Observed (`xo-stringtable2/utest/json_render.test.cpp`,
`DString-json-uses-size-not-nul`), rendering a 3-byte string `a\0b`:

```
"a\x00b"
```

`\x00` is a C/ppsink escape. JSON has no `\x`; it requires `\u0000`. A strict
parser rejects the document. Confirm with the observed case rather than
trusting this ticket:

```bash
.build/xo-stringtable2/utest/utest.stringtable2 "DString-json-uses-size-not-nul" -s
```

## Design (revised 2026-10-15, RC)

**Put the escape vocabulary on the sink, not in printjson.**  RC wants PpSink
to emit JSON eventually (`.xo-backlog/xo-printjson/issues/01`), so a
printjson-local escaper -- the original plan below -- would be a stopgap to
delete later.  This ticket builds the piece of 01 that 04 needs: 01's
"escape POLICY on the sink" (RC, 2026-09-20, its section "A third option"),
nothing more.

### Where the switch lives: `PpStyle`

`PpStyle` is "the presentation choices in force for a sink", held by value
on every `PpSink`, copied from defaults at construction, read by Prettifiers
through `sink.style()`:

```bash
cd "$(xo-sdlc --umbrella2)"
grep -n 'style()\|PpStyle style_' xo-ppsink/include/xo/ppsink/PpSink.hpp
grep -n 'FlatSink(const PpStyle' xo-ppsink/include/xo/ppsink/FlatSink.hpp   # a FlatSink takes one
```

It already holds the other JSON-relevant choice, `color_enabled` (which
`color.hpp` gates on).

- `PpStyle` gains an escape vocabulary, `{display, json}`, default
  `display`.  Only the spelling of characters with no short form differs:

  | byte | display (today) | json |
  |---|---|---|
  | `"` `\` newline cr tab `\b` `\f` | `\"` `\\` `\n` `\r` `\t` `\b` `\f` | same |
  | other C0 controls (0x00-0x1f) | `\xNN` | `\u00NN` |
  | DEL (0x7f) | `\x7f` | `\u007f` -- valid raw, but invisible in terminals, diffs and the page; escaping keeps it visible (RC, 2026-10-15) |
  | bytes >= 0x80 | raw | raw (UTF-8 passes through; JSON permits it) |

  So json escapes exactly the set display escapes, in JSON's spelling.
- A preset `PpStyle::json()`: json escapes, colour off.  Colour must be off
  for json: a coloured sink writes ANSI sequences into the document (01's
  constraint table; `xo-ppsink/utest/PpStyle.test.cpp` pins one).
  Enforcing the combination is 01's `JsonPpSink`; this ticket only supplies
  the preset.

### `Escape` becomes policy-aware -- sizing as well as emitting

`Escape` (`xo-ppsink/include/xo/ppsink/escape.hpp`) is all-static
`constexpr`.  `pair_char()` and `needs_hex()` -- WHICH bytes escape -- stay
as they are.  What takes the vocabulary:

- `str_copy()`: the fallback spelling (`\xNN` vs `\u00NN`);
- `str_size()`: the expanded size, 4 vs 6 bytes per fallback character.
  `PpState::put_with_escape` allocates its token from this
  (`xo-indentlog2/src/indentlog2/PpState.cpp:103`), so a size that ignored
  the vocabulary would under-allocate -- 01 already flagged this;
- `c_max_char_expand` (4): `FlatSink` reserves it in its stack buffer
  (`xo-ppsink/src/ppsink/FlatSink.cpp:76`) -- must cover json's 6.

```bash
cd "$(xo-sdlc --umbrella2)"
grep -rn 'str_copy\|str_size\|c_max_char_expand' xo-ppsink xo-indentlog2 --include=*.hpp --include=*.cpp | grep -v '/\.build'
```

### Both `put_with_escape` implementations pass the sink's choice

- `FlatSink::put_with_escape` (`xo-ppsink/src/ppsink/FlatSink.cpp:60`):
  `style_`'s vocabulary to `Escape`.
- `PrettySink::put_with_escape` forwards to `PpState::put_with_escape`
  (`xo-indentlog2/src/indentlog2/PrettySink.cpp:190`); `PpState` holds no
  `PpStyle`, so `PrettySink` passes its vocabulary as an argument.

The `put_with_escape(sv, quote_flag)` virtual keeps its signature.  `quot()`
and `unq()` are unchanged: they render through whatever sink they reach, so
they escape per that sink's style.

### printjson, until 01 lands

printjson still writes to `std::ostream`; `os << quot(s)`
(`xo-ppsink/include/xo/ppsink/quoted_ostream.hpp`) builds a `FlatSink` with
the DEFAULT style per call, i.e. display escaping.  So printjson needs a
json-styled bridge: a helper in xo-printjson -- e.g. `json_quot(s)`, whose
`operator<<` builds `FlatSink(PpStyle::json(), os.rdbuf())` -- replacing
every `quot(` in xo-printjson.  When 01 makes printjson write into a
(json-styled) sink, the helper goes: `quot()` through that sink is already
right.

**One decision to confirm when building:** the switch was first proposed
(2026-10-14, in conversation) as a format-level `PpStyle::format { text,
json }` implying colour off, rather than an escape field plus a preset.
Written here as escape field + preset because that keeps colour on one gate
(`color_enabled`) and matches 01's vocabulary ("escape POLICY"), with 01's
`JsonPpSink` enforcing the combination.  Revisit if a single `format` reads
better.

## Scope: re-derive it, do not trust the list below

**Re-derived 2026-10-15: wider than the list below.**  `quot()` now writes
every quoted token printjson emits -- string values, member names, map keys
(user data), object keys, type keys, `_error_` text -- 37 call sites in six
files:

```bash
cd "$(xo-sdlc --umbrella2)"
grep -rn 'quot(' xo-printjson/src xo-printjson/include | wc -l      # 37
grep -rln 'quot(' xo-printjson/src xo-printjson/include
#   JsonMembers.cpp type_keys.cpp PrintJson.cpp JsonPrintState.cpp
#   JsonObject.cpp, include/.../type_keys.hpp (a comment)
```

So the property-name question below is settled: keys take the same path.

Original scope, 2026-09-13:


At the time of writing the affected printers are the four registered by
`provide_string_printer` (`PrintJson.cpp:443-446`): `char *`, `char const *`,
`std::string`, `std::string_view` — plus anything delegating to them, which is
how `DString` is affected (`SetupStringtable2::provide_json_printers`).

```bash
grep -n 'provide_string_printer\|quot(' xo-printjson/src/printjson/PrintJson.cpp
```

Worth checking whether JSON *property names* take the same path
(`PrintJson.cpp:121` has a note about python requiring double quotes) — a key
containing a quote or backslash has the same problem and is easier to hit
accidentally than an embedded NUL, since `DDictionary` keys are user data.

## What the fix is not (2026-09-13 -- SUPERSEDED 2026-10-15, kept per CONVENTIONS rule 6)

*Superseded by "Design" above: the vocabulary moves onto the sink.  The
reasoning here still holds where it says `quot()`'s display escaping is right
for diagnostics -- that stays the default, and `quot()`'s callers outside
printjson see no change.  What changed is where JSON escaping lives: on a
json-styled sink, not in a printjson-local escaper.*


Not a change to `quot()`. Its escaping is correct for its job — readable
diagnostic output — and it is used far outside printjson. This wants a
json-specific escape local to xo-printjson, applied where `quot()` is called
now.

The JSON string grammar (RFC 8259 §7) is small: `"`, `\`, and the C0 controls
must be escaped; `\b \f \n \r \t` have short forms, everything else in
`U+0000`-`U+001F` takes `\u00XX`. Nothing else is required, so a faithful
implementation is short.

Note the UTF-8 question is separate and probably needs no work: JSON permits
raw multi-byte UTF-8, so only the controls need escaping. Do not widen this
ticket into a transcoding one without evidence that something needs it.

## Done when

Revised 2026-10-15 for the design above (the original four kept, three
added):

- `PpStyle` carries the escape vocabulary, default display; `PpStyle::json()`
  is json escapes with colour off; `FlatSink` and `PrettySink` honour it,
  each with a test of `\u0000`, `\u001b`, `\u007f` (and display unchanged:
  the existing `xo-indentlog2/utest/put_with_escape.test.cpp` stays green
  untouched);
- `Escape::str_size` and the buffer reserve are right for json's 6-byte
  expansion -- a test of a string of only control characters through
  `PrettySink`, json style (the case that under-allocates if sizing ignores
  the vocabulary);
- nothing in xo-printjson calls `quot(` (`grep -rn 'quot(' xo-printjson/src
  xo-printjson/include` finds nothing);
- an embedded NUL renders as `\u0000`, and the case in
  `xo-stringtable2/utest/json_render.test.cpp` is updated to that expectation
  (it carries a note saying a failure there means *this* ticket landed, not
  that DString broke)
- a quote and a backslash inside a string survive a round trip through
  `json.loads`, as do the same characters in a property name
- `PrintJson.cpp:367`'s TODO is gone rather than reworded
- a test in xo-printjson's own utest, so the coverage does not live only in a
  subsystem that happens to have found it

## Provenance

Found 2026-09-13 while implementing `AReflectable` for `DString`
(`.xo-backlog/reflectable2/issues/05-per-type-opt-in.md`). Not fixed there:
the rendering defect belongs to every string printjson emits, not to DString.
