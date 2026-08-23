# 08 — mine xo-indentlog's docs for xo-indentlog2

Status: open
Type: task
Milestone: ppsink-migration

Progress: find xo-indentlog/docs -name '*.rst' | wc -l

## Why this exists

`xo-indentlog` has **zero dependents** and has had since 2026-08-13:

```bash
xo-deps --users-of=xo-indentlog --format=names
#   0 subsystems
```

so nothing stops it being deleted except that its documentation has not been
carried across. RC 2026-08-23: **keep xo-indentlog until this is done.** That is
the recorded reason `ppsink-migration`'s "deleted, or explicitly kept with a
recorded reason" asks for — and this ticket is what stops "for now" becoming
permanent by inattention.

## What is there

Measured 2026-08-23:

```bash
find xo-indentlog/docs  -type f | wc -l   #  14 files, 372K
find xo-indentlog2/docs -type f | wc -l   #   2 files,  12K
```

The eight reference documents, none of which has an xo-indentlog2 counterpart:

| doc | subject | v2 equivalent to write against |
|---|---|---|
| `glossary.rst` | vocabulary of the pretty-printer | still the vocabulary; `xo::pp` names differ |
| `scope-reference.rst` | `scope` / nesting / `XO_SCOPE` macros | `xo-ppsink/scope.hpp` — same idea, new sink |
| `ppstr-reference.rst` | the `ppstr` layout primitives | `PpSink` begin/split/end, `pretty_struct` |
| `ppconfig-reference.rst` | config knobs | `PpConfig`, `PpStyle`, `ArenaConfig` |
| `logging_intro.rst` | how to log | `scope` + `PrettySink` install |
| `logging_impl.rst` | how logging works inside | `LogBuffer`, `LogStreambuf`, `LogState` |
| `pretty_impl.rst` | how pretty-printing works inside | `PpState`, `PpToken`, the fits/break algorithm |
| `install.rst` | build/install | v2 equivalent likely trivial |

## Not a copy job

The v1 docs describe an ostream-and-`ppindentinfo` design that v2 deliberately
replaced: v2 emits a token stream to a `PpSink` and decides breaks downstream,
with no trial-fit pass (`xo-ppsink/pretty_struct.hpp` states that difference
explicitly). So `pretty_impl.rst` and `ppstr-reference.rst` describe mechanics
that **no longer exist**. What carries over is the vocabulary and the worked
examples; what does not is most of the implementation narrative.

Expect to write, not port, for `pretty_impl` and `ppstr-reference`; expect to
port with renaming for `glossary`, `scope-reference`, `logging_intro`.

## Done when

- xo-indentlog2 has counterparts for the docs judged worth keeping — the list
  above is a starting inventory, not a mandate to carry all eight
- for each v1 doc NOT carried across, a line here saying why
- `xo-indentlog` can then be deleted, which also removes 20 files from
  ostream-containment's counter C (`xo-ppsink/issues/15`)

## Side effect worth knowing

While xo-indentlog stays, counter C carries its 20 files as permanent
non-work — 19% of the 105 remaining as of 2026-08-23. See the note in
`.xo-backlog/milestones/ostream-containment.md`; that milestone's counters
cannot reach 0 until this ticket closes or C gains an exclusion for
xo-indentlog the way it will need one for xo-ppsink's own machinery.
