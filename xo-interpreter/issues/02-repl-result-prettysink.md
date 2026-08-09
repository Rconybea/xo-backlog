# 02 — REPL prints results flat, though it already holds a PrettySink

Status: open
Type: feature

The Schematika REPL prints each result through an ostream inserter, which never
breaks a line:

```cpp
/* xo-interpreter/src/interpreter/Schematika.cpp, print-value site */
cout << "scm result:" << endl;
cout << value << endl;
//pps.pretty(value);          <- commented out, and pps is right there
```

`value` is a `gp<Object>`. Since
`.xo-backlog/xo-alloc/issues/01-object-pretty-ppsink.md` (done 2026-08-08) that
inserter goes through `Object::pretty(PpSink&)` over a `FlatSink` — and a
FlatSink renders every split as its flat spaces and never breaks. So a long or
deeply nested result prints as one unbroken line, however wide the terminal.

The REPL already constructs a `PrettySink` (`pps`), and the commented-out line
shows the intent was there.

## Why now, when it was not worth it before

Before the conversion, rendering through `pps` would have bought nothing: every
Object arrived as one atomic string token, so there were no break points to
use. The conversion put real structure in — `List` separates elements with
`split(1)`, `pretty_struct` fields are breakable — so a PrettySink now has
something to work with.

Deliberately not done as part of xo-alloc/01: that ticket's contract was to
leave rendered output unchanged, and this changes it.

## Questions to settle

- **Margin.** Fixed, or the terminal width? The REPL uses replxx, which knows
  the terminal size; a fixed margin is simpler but wrong on a wide terminal.
- **Does the result print differ from diagnostics?** A user reading a value may
  want different wrapping from a debug log line. If so this is a PpConfig
  choice at the print site, not a global one.
- **Arena naming.** Two PrettySinks sharing an ArenaConfig name interfere, and
  the symptom is wrong indentation in whichever runs second — documented in
  `xo-alloc/utest/ObjectStatistics.test.cpp`. A REPL creating one per result
  must vary the name or reuse a single sink.

**Files:**
- `xo-interpreter/src/interpreter/Schematika.cpp` — the print-value site, and
  the `#include <xo/alloc/alloc_ostream.hpp>` that currently supplies `<<`
- `xo-indentlog2` — where `PrettySink` and `PpConfig` live

**Done when:**
- a result too wide for the margin prints broken across lines rather than
  running off
- short results are unchanged
- pinned by a test at a known margin, asserted against measured output rather
  than predicted output

## Notes

Check whether anything parses the REPL's stdout before changing it. Breaking a
line is a visible output change, and "prints on one line" is the kind of thing
a script quietly depends on.
