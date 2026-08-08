# 04 — escape.hpp: short escapes for tab (and friends), not \xNN

Status: open
Type: task

`xo::pp::Escape` gives four characters a two-character short form and sends every
other control character to `\xNN`:

```cpp
/* xo-ppsink/include/xo/ppsink/escape.hpp -- Escape::pair_char() */
case '\\': return '\\';
case '"':  return '"';
case '\n': return 'n';
case '\r': return 'r';
default:   return '\0';     /* -> needs_hex() -> \xNN */
```

So tab renders as `\x09` where most readers would expect `\t`:

```
xo::pp::quot("a\tb")  ->  "a\x09b"
xo::pp::quot("a\nb")  ->  "a\nb"      (already short, and correct)
```

Add `\t` at minimum. `\b` (0x08) and `\f` (0x0c) are the conventional companions
and cost one line each; whether xo wants them is a judgement call -- they are far
rarer and arguably clearer as `\x08` / `\x0c`.

## Why it matters

Surfaced twice during the indentlog->ppsink migration:

- `xo-object`'s `String::display()` (`String.cpp:127`) renders a Schematika
  string value as `quot(c_str())` -- a language-level literal, where `\t` reads
  and round-trips and `\x09` may not (see `.xo-backlog/xo-object/issues/01`).
- `xo-websock`'s `WebsocketSink` hand-builds a JSON envelope with `quot`.
  **`\xNN` is not valid JSON either** -- JSON allows only
  `\b \f \n \r \t \" \\ \/ \uXXXX`. Legacy was also wrong there (it emitted raw
  control characters, which JSON forbids outright), so this is not a regression,
  but adding `\t` fixes one of the two JSON-invalid cases for free.

## Constraint that must survive

**ESC (0x1b) must keep escaping.** From `needs_hex()`'s comment:
`PpState::count_visible_chars` treats a raw ESC as the start of a (zero-width)
colour escape, so a raw ESC inside a string token makes its visible length
undercount and breaks line fitting. Whatever changes, ESC must not reach the sink
raw.

Bytes >= 0x80 must also keep passing through untouched, so UTF-8 survives.

## Watch out for

`Escape::str_size()` and `Escape::str_copy()` must agree exactly -- size is
computed in a separate pass before allocation, so a character that copies as 2
bytes but sizes as 4 corrupts the buffer. Both consult `pair_char()`, so adding a
case there should keep them in step, but the tests should pin it.

**Files:**
- `xo-ppsink/include/xo/ppsink/escape.hpp` — `pair_char()`, `needs_hex()`,
  `str_size()`, `str_copy()`
- `xo-ppsink/utest/escape.test.cpp` — where the escape table is pinned
- `xo-ppsink/utest/hex.test.cpp` — unaffected (hex has its own formatting)

**Done when:**
- tab renders `\t`
- a decision is recorded on `\b` / `\f`
- ESC and other control characters still escape; UTF-8 still passes through
- `str_size()` and `str_copy()` agree for every character, pinned by test

## Notes

Deliberately does not settle the JSON question -- valid JSON also needs
`\uXXXX` for the remaining control characters, which is a bigger change and only
matters where ppsink output *is* JSON. See
`.xo-backlog/xo-printjson/issues/01-ppsink-api.md`.
