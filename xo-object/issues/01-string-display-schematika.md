# 01 — String::display(): make the printed form Schematika-specific

Status: open
Type: task

`xo::obj::String` is a Schematika string *value*. Its printed form is currently
whatever xo's generic diagnostic quoting produces:

```cpp
/* xo-object/src/object/String.cpp:127 */
String::display(std::ostream & os) const {
    os << quot(c_str());
}
```

`quot` is a **logging** facility: it escapes enough to keep a value unambiguous
inside a log line. That is not the same contract as "print a Schematika string
literal", which ideally round-trips through xo-reader.

## What changed, and why this is open now

The ppsink migration (2026-08-07) swapped legacy `xo::print::quot` for
`xo::pp::quot`. Measured difference, control characters only:

| input | legacy | ppsink |
|---|---|---|
| `a b`, `say "hi"`, `a\b`, newline, CR | identical | identical |
| **tab** | raw `<TAB>` inside the quotes | `\x09` |
| other control chars | raw | `\xNN` |
| UTF-8 | unchanged | unchanged |

Accepted deliberately: escaping beats emitting a raw control character into a
quoted literal. But it made the underlying question visible -- neither form is
necessarily what Schematika wants.

Nothing pins this: no test asserts on `String::display()` output, so the change
was invisible to the suite.

## The question to settle

Does `String::display()` owe a **round-trippable Schematika literal**, or is it
a diagnostic rendering that merely needs to be unambiguous?

- If round-trippable: the escape set is Schematika's, not ppsink's. It must match
  what xo-reader accepts -- check the tokenizer's string-literal handling
  (`xo-tokenizer`, and see its escape diagnostics: "expecting one of n|r|\"|\\
  following escape \\"). Note that set is currently **narrower** than what ppsink
  emits: xo-reader appears to accept `\n \r \" \\` and *not* `\xNN`, so today's
  output does not round-trip.
- If diagnostic: leave it on `quot`, and `.xo-backlog/xo-ppsink/issues/04`
  (short escapes for tab) is the whole fix.

The narrowness of the reader's escape set is the strongest argument that these
are two different jobs and `String` should own its own rendering.

## Interaction with other tickets

- `.xo-backlog/xo-ppsink/issues/04` — adding `\t` narrows the gap but does not
  close it; `\xNN` still is not Schematika syntax.
- `.xo-backlog/xo-alloc/issues/01` — `display()` becomes `pretty(PpSink&)` across
  the Object hierarchy. **Sequence that first**, or this rendering gets written
  twice. Whatever escape policy is chosen should land in the `pretty()` body.

**Files:**
- `xo-object/src/object/String.cpp:127` — the rendering
- `xo-tokenizer/include/xo/tokenizer/tokenizer.hpp` — the reader's escape set,
  for the round-trip question
- `xo-ppsink/include/xo/ppsink/escape.hpp` — what `quot` currently does

**Done when:**
- a decision is recorded on round-trip vs diagnostic
- if round-trip: `String`'s printed form re-reads to an equal `String`, pinned by
  a test that goes through xo-reader
