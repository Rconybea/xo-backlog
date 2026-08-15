# 02 — move activation_record's ostream support to activation_record_ostream.hpp

Status: open
Type: refactor
Milestone: ostream-containment
Progress: grep -c 'operator<<(std::ostream' xo-jit/include/xo/jit/activation_record.hpp

`xo-jit/include/xo/jit/activation_record.hpp:76` already carries the intent as a
comment:

```cpp
// TODO: move to activation_record_ostream.hpp; prefer PpSink-based infra
```

This ticket is that TODO, promoted so it is visible from the milestone. It is a
textbook instance of `ostream-containment`'s population B — a header-defined
printer doing `os << xo::pp::xtag(..)` in a public header.

## What has to move

Exactly one inserter, and the include that exists only to serve it:

| what | where |
|---|---|
| `operator<<(std::ostream&, const runtime_binding_detail&)` | `activation_record.hpp:77-86` (inline) |
| `#include <xo/ppsink/tag_ostream.hpp>` | `activation_record.hpp:14` |
| the 4-line comment explaining why `xtag` is qualified | `activation_record.hpp:10-13` |

The include and the comment go **with** it: `xo::pp::xtag` appears nowhere else
in the header.

```bash
grep -n 'xo::pp\|xtag\|tostr\|scope' xo-jit/include/xo/jit/activation_record.hpp
#   :10,:12  the comment
#   :80-:82  inside the inserter
#   :222     'runtime_binding_detail' as a map value -- not a pp use
```

`activation_record` itself declares no inserter, and neither does
`runtime_binding_path`. This is the whole ostream surface of the header.

## Nothing streams it today (measured 2026-08-15)

The inserter has no in-tree caller. `runtime_binding_detail` is referenced
exactly once outside its own header:

```bash
grep -rn 'runtime_binding_detail' --include=*.hpp --include=*.cpp . \
  | grep -v '/\.build/' | grep -v 'activation_record\.\(hpp\|cpp\)'
#   xo-jit/src/jit/MachPipeline.cpp:882
```

and that site (`MachPipeline.cpp:882-889`) binds a `const
runtime_binding_detail *` and reads `->llvm_type_` / `->llvm_addr_` from it. It
never streams it.

So the move cannot break a caller, because there is no caller. That also means
the *cheap* version of this ticket — delete the inserter outright rather than
rehousing it — is on the table; see **Done when** below.

## Dropping the include is the part that could bite, and does not

`activation_record.hpp` is included by `MachPipeline.hpp:13`, so removing
`tag_ostream.hpp` from it withdraws `xtag` and the ppsink inserters from every
TU downstream. That is precisely the failure mode this tree hit twice on
2026-08-13 (a converted header stops propagating, and an unrelated file that had
been free-riding fails to compile). Falsification, per `CONVENTIONS.md` rule 3:

```bash
for f in $(grep -rlE '\bxtag\(' --include=*.hpp --include=*.cpp xo-jit/ | grep -v '/\.build/'); do
  grep -q 'ppsink/tag' "$f" || echo "$f"
done
#   xo-jit/src/jit/MachPipeline.new.cpp
#   xo-jit/src/jit/MachPipeline.orig.cpp
```

Two files, and **neither is built** — `xo-jit/src/jit/CMakeLists.txt:7,9` lists
only `MachPipeline.cpp` and `activation_record.cpp`. Every compiled xo-jit TU
that names `xtag` includes a ppsink tag header itself. So the include can be
withdrawn without adding one anywhere.

## The comment at :10-13 needs a decision, not a copy-paste

It reads:

> *xtag is written out qualified below: this is a public header and the tag
> argument types live in namespace xo, so an unqualified call would also reach
> legacy `xo::xtag` by ADL in any consumer that still sees xo-indentlog.*

That hazard is now historical — no subsystem reaches legacy xo-indentlog:

```bash
xo-deps --users-of=xo-indentlog --format=names -q     # empty (still, 2026-08-15)
```

Qualifying is still correct style for a public header, but the *reason given* no
longer applies. Either restate the reason or drop the comment; do not carry it
across verbatim as though it were still live. (Same lesson as the workaround
comments cleaned up in `.xo-backlog/xo-reader2/issues/02`: an explanatory comment
has the lifetime of the thing it explains.)

## Shape to follow

`xo-webutil/include/xo/webutil/webutil_ostream.hpp:1-25` is the model the
milestone names, and its header comment carries the reasoning worth reproducing
in spirit — PpSink is the intended path inside xo; the ostream inserter is the
paved road for newcomers, tests and REPL output.
`xo-alloc/include/xo/alloc/alloc_ostream.hpp` is the second precedent.

Note the milestone's open naming question: ten headers in the tree spell this
`_iostream.hpp` rather than `_ostream.hpp`. The TODO and this ticket both say
`_ostream.hpp`; if the milestone settles the other way, this is one of the files
to rename.

## Files

- `xo-jit/include/xo/jit/activation_record.hpp:10-14,76-86` — what moves out
- `xo-jit/include/xo/jit/activation_record_ostream.hpp` — new
- `xo-jit/include/xo/jit/MachPipeline.hpp:13` — the downstream includer
- `xo-jit/src/jit/CMakeLists.txt:7,9` — shows which sources are real
- `xo-webutil/include/xo/webutil/webutil_ostream.hpp` — the pattern

## Done when

- `activation_record.hpp` names no `std::ostream` and includes no ppsink
  ostream header
- the inserter is either in `activation_record_ostream.hpp` or **deleted** —
  decide against `.xo-backlog/xo-ppsink/issues/10-verify-inserters-unused.md`,
  whose whole question is whether the contained inserters have any live caller.
  This one demonstrably has none, so it is a candidate for deletion rather than
  containment, and rehousing it first would be work done twice.
- if it survives, a `Prettifier<runtime_binding_detail>` exists beside it so the
  type stops relying on ppsink's silent `operator<<` fallback
  (`xo-ppsink/include/xo/ppsink/pretty.hpp:29-32`). This is what the TODO's
  "prefer PpSink-based infra" is asking for, and it is the part with actual
  design content — the other steps are file moves.
- `xo-build --sweep` still reads
  `62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`

## Notes

**Four scratch copies sit beside the real files and are not built.**

```bash
ls xo-jit/include/xo/jit/activation_record.{new,orig}.hpp \
   xo-jit/src/jit/{activation_record,MachPipeline}.{new,orig}.cpp
grep -n 'activation_record\|MachPipeline' xo-jit/src/jit/CMakeLists.txt
#   :7  MachPipeline.cpp
#   :9  activation_record.cpp     <- .new/.orig appear nowhere
```

All four `.cpp` copies include `activation_record.hpp`, so they are the only
things a move here can break — and because they are not compiled, **no sweep
will tell you**. They were edited during the `tostr` → `tostr0` rename on
2026-08-15 for the same reason (found by a use-based grep, changed, never
compiled). Worth deciding whether they are still wanted before this ticket adds
another unverified edit to them; if they are scratch, deleting them removes a
standing trap.
