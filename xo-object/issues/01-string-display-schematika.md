# 01 — String::display(): make the printed form Schematika-specific

Status: open
Type: task
Milestone: ppsink-migration

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
  what xo-reader accepts -- see the corrected measurement below.
- If diagnostic: leave it on `quot`, and `.xo-backlog/xo-ppsink/issues/04`
  (short escapes for tab) is the whole fix.

### Corrected 2026-08-08: the reader accepts `\t` too

This ticket previously read:

> xo-reader appears to accept `\n \r \" \\` and *not* `\xNN`

**That set was wrong, and it was wrong in the way `CONVENTIONS.md` warns about:
it was read off the tokenizer's own error message rather than its code.** The
message under-reports what the switch actually handles:

```bash
# five cases, including 't'
sed -n '514,541p' xo-tokenizer/include/xo/tokenizer/tokenizer.hpp | grep -E '^\s+case'
#   case '\\':  case 'n':  case 't':  case 'r':  case '"':

# but the default branch's diagnostic omits t (tokenizer.hpp:538)
#   "expecting one of n|r|\"|\\ following escape \\"
```

Already pinned by an existing test — `xo-tokenizer/utest/tokenizer.test.cpp:191`
feeds `"tab to the right [\t]..."` and expects a real tab in the token.

Why the wrong reading was plausible: the diagnostic is the only place the
accepted set is written down as a *set*, so it reads like the specification. It
is instead a hand-maintained string that drifted from the switch beside it.

**Consequence for this ticket.** As of `xo-ppsink/issues/04`, ppsink emits short
forms for `\\ " \n \r \t \b \f`. The reader accepts `\\ n t r "`. So every
escape the reader knows is now emitted in a form it can read back — a string
containing only tab, newline, CR, quote and backslash **does** round-trip today,
where before this change tab went out as `\x09` and the reader rejected it.

What still does not round-trip: `\b`, `\f`, and every remaining control
character (`\xNN`). So the round-trip-vs-diagnostic question is still open, but
the gap is now narrow and specific rather than "the reader is much narrower".

The tokenizer's stale diagnostic is a separate small bug: it should name `t`.

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
