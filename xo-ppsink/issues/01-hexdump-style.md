# 01 — hexdump style for hex_view (offsets + ascii gutter)

Status: open
Type: feature
Milestone: ppsink-migration

`xo::pp::hex_view` (`xo-ppsink/include/xo/ppsink/hex.hpp`) currently renders a
byte range as a bracketed run, breaking at 16-byte row boundaries:

```
[00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f
  10 11 12 13 14 15 16 17 18 19 1a 1b 1c 1d 1e 1f
  20 21 22 23 24 25 26 27]
```

That shape was chosen to stay byte-identical to the legacy `xo::hex_view` it
replaced whenever the range fits one line. It is a faithful port, not the best
tool for the job it actually gets reached for.

**The job is reading memory layout** — where a struct's fields land, what a
`flatstring`'s capacity buffer holds past its size, whether padding is what you
expected. For that, the classic `xxd`/`hexdump -C` layout is decisively better:

```
0000  00 01 02 03 04 05 06 07  08 09 0a 0b 0c 0d 0e 0f  |................|
0010  10 11 12 13 14 15 16 17  18 19 1a 1b 1c 1d 1e 1f  |................|
0020  20 21 22 23 24 25 26 27                           | !"#$%&'|
```

A byte offset column lets you name an address without counting; the ascii
gutter reads as text without the `(c)` annotation doubling the width of every
byte; the 8-byte half-row gap makes word boundaries visible at a glance.

## Shape

Add a third `hexstyle` enumerator rather than a new type. The enum already
exists and is already the selector:

```cpp
enum class hexstyle {
    bare,        // [68 65 6c]
    with_char,   // [68(h) 65(e) 6c(l)]
    dump,        // offsets + ascii gutter, one row per line
};
```

The 16-byte grouping is already in place (`hex_view::c_bytes_per_row`, with
`split(1)` only at row boundaries), so the row boundaries this needs are
exactly the ones already being emitted. This is additive, not a rewrite.

## Decisions to make

- **Forced breaks.** `bare`/`with_char` use `split(1)`, so a short range stays
  on one line. A dump is only legible one row per line, so `dump` presumably
  wants `newline()` instead — which forces every enclosing group to break too.
  Confirm that is acceptable when a dump appears as a tag value inside a
  larger structure.
- **Offset width and base.** Fixed 4 hex digits, or scaled to the range size?
  Offsets relative to the start of the range, or the actual address?
- **Row width.** Fixed at `c_bytes_per_row`, or configurable? A configurable
  width means the split points can no longer be decided at print time from a
  constant.
- **Whether `dump` still brackets.** The `[`/`]` framing earns its keep for an
  inline run; for a multi-row dump it may just be noise.

## Not in scope

Changing `bare` or `with_char`. Their current rendering is pinned by
`xo-ppsink/utest/hex.test.cpp` (flat text + token stream) and
`xo-indentlog2/utest/hex.test.cpp` (rendered wrapped form), and the
byte-compatibility with legacy `xo::hex_view` is deliberate — see the header
comment in `hex.hpp`.

**Files:**
- Modify: `xo-ppsink/include/xo/ppsink/hex.hpp` — `hexstyle`, `Prettifier<hex_view>`
- Test: `xo-ppsink/utest/hex.test.cpp` — flat/token-stream cases
- Test: `xo-indentlog2/utest/hex.test.cpp` — the rendered rows; this is the
  only place a real `PrettySink` is available, so it is the only place the
  row layout can actually be asserted

**Done when:**
- `hexstyle::dump` renders offsets + ascii gutter
- `bare` and `with_char` output is unchanged (existing tests still pass
  untouched)
- the dump layout is pinned against measured output, not predicted output

## Notes

Measure before asserting. The `pretty_struct` work burned a long stretch
chasing an indent bug that did not exist, because expectations were written
from prediction rather than from observed output — and because Catch2 indents
both sides of a multi-line string comparison by 2 when it displays them.
Render first, read the actual bytes, then write the assertion.
