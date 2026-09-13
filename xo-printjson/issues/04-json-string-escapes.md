# 04 — string rendering escapes with ppsink's vocabulary, not JSON's

Status: open
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

## Scope: re-derive it, do not trust the list below

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

## What the fix is not

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
