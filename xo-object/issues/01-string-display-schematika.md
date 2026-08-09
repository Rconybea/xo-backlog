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
Filed as `.xo-backlog/xo-tokenizer/issues/02`.

### Corrected 2026-08-09: round-trip was not "never available" — it was LOST

This ticket reads as though Schematika never had a round-trippable printed
form and the question is whether to acquire one. That is wrong. **Legacy
round-tripped, and the ppsink migration traded it away.**

The reader takes any unrecognised character in a string body verbatim:

```bash
sed -n '543,545p' xo-tokenizer/include/xo/tokenizer/tokenizer.hpp
#   default:
#       tk_text.push_back(*ix);
```

So a raw control character inside a literal is accepted. Legacy `quot` emitted
control characters raw — see this ticket's own table, tab as raw `<TAB>`,
others raw — so print/read was a round trip. ppsink escapes them to `\xNN`,
which the reader rejects, and that is what broke it.

The trade was correct, and it is not reversible: `xo-ppsink/escape.hpp` **must**
escape ESC, because `PpState::count_visible_chars` treats a raw ESC as the
start of a zero-width colour escape, undercounts the token's visible width and
breaks line fitting. Emitting raw control characters again would reintroduce
that.

So the option space is three, not two:

| | round-trips | cost |
|---|---|---|
| **A** diagnostic — keep `\xNN` | no, for control chars | none; record and close |
| **B** raw control characters, as legacy | yes | breaks ppsink line fitting — ruled out |
| **C** teach the reader `\xNN` | **yes, for every byte** | one `case 'x':` in the tokenizer |

**Decision (RC, 2026-08-09): C.**

C is the only option that closes the gap rather than narrowing it, and it is
cheap because the printer already emits exactly the form the reader would gain:
one more case in the same switch as the five existing escapes, plus two hex
digits. Printer and reader then agree for every byte, not just the five
characters that happen to have short forms.

Note this is a **Schematika language change** — `\xNN` becomes valid surface
syntax — not merely a printing fix.

Sequence it with `.xo-backlog/xo-tokenizer/issues/02`: same function, same
switch, and both are the same underlying defect — the reader's escape set, and
its own description of that set, having drifted from what the printer emits.

Once C lands, `String::pretty()` needs no change at all: its current
`sink.pp(quot(c_str()))` becomes round-trippable by virtue of the reader
catching up. The test this ticket asks for — print, re-read, compare — is then
the thing that pins the two together.

## Interaction with other tickets

- `.xo-backlog/xo-ppsink/issues/04` — adding `\t` narrows the gap but does not
  close it; `\xNN` still is not Schematika syntax.
- `.xo-backlog/xo-alloc/issues/01` — **DONE 2026-08-08.** This ticket's
  prerequisite is cleared: `display()` is gone from the Object hierarchy, and
  the rendering now lives in `String::pretty(PpSink&)`. It was converted
  byte-identically on purpose, so the escape-policy question is untouched and
  this ticket is now unblocked rather than partly done.

  Whatever policy is chosen lands in that `pretty()` body — and it now has a
  PpSink in hand, so a Schematika literal can be emitted as structure rather
  than a pre-quoted string if that turns out to matter.

**Files:**
- `xo-object/src/object/String.cpp` — the rendering, now `String::pretty()`
  (was `display()` at `:127`; the line moved during the conversion — do not
  trust that number)
- `xo-tokenizer/include/xo/tokenizer/tokenizer.hpp` — the reader's escape set,
  for the round-trip question
- `xo-ppsink/include/xo/ppsink/escape.hpp` — what `quot` currently does

**Done when:**
- ~~a decision is recorded on round-trip vs diagnostic~~ — done 2026-08-09,
  option C above
- the reader accepts `\xNN`, so what `xo::pp::quot` already emits re-reads
- `String`'s printed form re-reads to an equal `String`, pinned by a test that
  goes through xo-reader — including a control character, since that is the
  case the whole ticket is about
- `\b` and `\f`: decide whether the reader gains them too, or whether the
  printer should stop emitting short forms it cannot read back. Either closes
  the gap; emitting `\b` while accepting only `\xNN` would not.
