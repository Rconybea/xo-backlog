# 04 — escape.hpp: short escapes for tab (and friends), not \xNN

Status: fixed 2026-08-08
Type: task
Milestone: ppsink-migration

## Resolution (2026-08-08)

`Escape::pair_char()` gained three cases — `\t`, `\b`, `\f` — bringing the
short-form set to `\\ \" \n \r \t \b \f`. That is exactly JSON's backslash-pair
set minus `\/`.

**Decision on `\b` / `\f`: included.** The ticket left this open. Taken because
they are the conventional C/JSON companions and cost one line each; the case
against (no in-tree consumer asks for them) did not outweigh having one
consistent escape vocabulary. Every remaining control character, `\v` (0x0b)
included, still goes to `\xNN`.

`c_max_char_expand` is unchanged at 4 — `\xNN` is still the widest expansion,
since the set of hex-escaped characters only shrank.

### The `str_size`/`str_copy` hazard did not materialise, and is now pinned

Both methods branch on `pair_char()`, so adding cases there kept them in step by
construction rather than by discipline. Pinned on both paths:

- `xo-ppsink/utest/escape.test.cpp` — the `escape_str()` helper REQUIREs
  `str_copy` wrote exactly `str_size().size` bytes, and compile-time
  `static_assert`s fix the widths (`\t`/`\b`/`\f` = 2, `\v` = 4).
- `xo-indentlog2/utest/put_with_escape.test.cpp` — the whole short-form set
  through `PrettySink`. This is the path that matters: the `PpStringToken` is
  sized first and filled after, so a disagreement corrupts the buffer rather
  than merely misformatting. `\v` is asserted alongside, so the set staying
  bounded is pinned too, not just the additions.

### Verified

```bash
SUBS=$(xo-build --list | tr ' ' '\n' | grep '^xo-' \
       | while read s; do [ -f "$s/CMakeLists.txt" ] && echo "$s"; done)
xo-build -q --configure --with-utests --with-examples --build --install $SUBS
for s in $SUBS; do ctest --test-dir $s/.build; done
```

61 subsystems build clean; 34 suites pass, 1 fails — `xo-jit machpipeline.fptr`,
the documented pre-existing failure (`.xo-backlog/xo-jit/issues/01`).

Exactly one downstream expectation pinned the old rendering
(`put_with_escape.test.cpp:56`, `a\x09b`). It was watched to fail before being
updated, which confirms that test actually catches this class of change. Found
with:

```bash
grep -rn 'x09\|x08\|x0c' --include=*.cpp --include=*.hpp xo-*/ \
  | grep -v '/\.build/' | grep -v '^xo-indentlog/'
```

Not run: `nix-build ci.nix -A xo-ppsink`. This change adds no dependency and
alters no install surface — it is a behaviour change inside an existing header —
so the check that nix uniquely provides (installed package config exercised as a
real consumer) has nothing new to exercise here.

### Consequence the ticket did not anticipate: tab now round-trips

This was scoped as a readability fix. It is more than that, because
`xo-object/issues/01`'s claim that xo-reader accepts only `\n \r \" \\` was
wrong — the tokenizer accepts **five** escapes including `t`
(`xo-tokenizer/include/xo/tokenizer/tokenizer.hpp:514-541`; pinned by
`xo-tokenizer/utest/tokenizer.test.cpp:191`). The claim had been read off the
tokenizer's error message (`tokenizer.hpp:538`), which omits `t`.

So a string containing tab now survives `String::display()` → xo-reader, where
`\x09` was rejected outright. Correction recorded in
`.xo-backlog/xo-object/issues/01-string-display-schematika.md`.

`\b` and `\f` are *not* in the reader's set, so they do not round-trip — the
JSON rationale and the Schematika rationale pull in slightly different
directions, which `xo-object/issues/01` still has to settle.

**Follow-on, not filed:** `tokenizer.hpp:538`'s diagnostic should name `t`.

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
