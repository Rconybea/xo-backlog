# 08 — `default_style_guard` is not thread-safe: put the style stack in `PpState`

Status: diagnosed 2026-08-09
Type: bug + design

Raised by RC 2026-08-09, on reading the `PpStyle` change from the same day.

## The defect

`xo::pp::PpStyle::default_style()` is a singleton
(`xo-ppsink/include/xo/ppsink/PpStyle.hpp`), and `default_style_guard`
read-modify-restores it:

```cpp
explicit default_style_guard(const PpStyle & style)
    : orig_{PpStyle::default_style()} { PpStyle::default_style() = style; }
~default_style_guard() { PpStyle::default_style() = orig_; }
```

Being a function-local static, its *initialization* is thread-safe. Its
*mutation* is not synchronized at all. Two threads rendering concurrently give:

- a data race on the `PpStyle` object — undefined behaviour, not merely a wrong
  colour;
- last-restore-wins, so a guard can restore a value another thread installed;
- a torn read by any `PpSink` or `PpConfig` constructed meanwhile, since both
  copy `default_style()` at construction.

**This is not hypothetical for xo**, because the sink model is already
per-thread: the active sink is `thread_local`
(`xo-ppsink/src/ppsink/LogState.cpp:38`), and `ThreadPrettySink::thread_install_once`
exists precisely so each thread renders through its own. A process-wide style
under a per-thread sink is the inconsistency.

**Scope narrowed 2026-08-09, same day this was filed.** As written, this ticket
named a second instance: `color_enabled_guard` over
`color_config::color_enabled`, the same read-modify-restore over the same kind
of global. RC removed it immediately rather than carrying it — `color_enabled`
moved into `PpStyle`, `color_guard` reads `sink.style().color_enabled` (it
already had the sink), and `color_config` and `color_enabled_guard` are deleted.
See `.xo-backlog/xo-printable2/issues/01`.

So **`PpStyle::default_style()` and `default_style_guard` are now the only
remaining instance**, which makes this ticket smaller and its subject sharper:
one singleton, mutated by one guard, reachable only because the convenience
entry points build their own sinks.

**What is already fine:** `PpSink` holds its `PpStyle` **by value**
(`PpSink.hpp`), and `PpConfig` likewise. A sink's own style is therefore
per-sink and race-free. The problem is confined to the process-wide default and
the one guard that moves it — which exists only to reach the convenience entry
points (`tostr()`, the `operator<<` bridge in `tag_ostream.hpp`) that build
their own sink internally and so cannot be handed a style.

## RC's proposal

There is already an explicit place for pretty-printer-scoped state: **`PpState`**
(`xo-indentlog2/include/xo/indentlog2/print/PpState.hpp`), owned per PrettySink
(`PrettySink.cpp:14`, `pps_{cfg}`). Put a **style stack** there, with push/pop
encoded as `PpToken`s.

Two things fall out:

- **Thread safety by construction.** Per-sink state, no global to race on.
- **Nesting with a defined extent.** A printer pushes a style for a subtree and
  pops at its end — which is the "nested formatting context" of
  `.xo-backlog/xo-ppsink/issues/07`, arriving as a mechanism rather than a
  second one.

### There is room in the token encoding

`PpTokenFlags` (`xo-indentlog2/include/xo/indentlog2/print/PpTokenType.hpp`)
uses a 2-bit type field:

```
k_string = 0x01, k_begin = 0x02, k_split = 0x03, k_end = 0x04
k_type_mask = (k_string | k_begin | k_split | k_end)     // == 0x07
```

The mask is `0x07` but only values 1..4 are used, so **5, 6 and 7 are free** —
`k_push_style` and `k_pop_style` fit without widening the mask or disturbing
`k_size_established` / `k_fits` / `k_forced` above it. Worth confirming by
reading the token-packing code before relying on it.

## Questions the design has to answer

- **Where does the pushed `PpStyle` live?** A token is small and lives in an
  arena (`PpState::tk_buffer_`). Either the token carries the style inline
  (~20 bytes today, and it will grow — issue 07 wants precision, issue 03 wants
  max-items) or it carries an index into a per-render style table. The second
  keeps tokens small and makes "same style as the enclosing scope" free.

- **`PpState` is in xo-indentlog2; `FlatSink` has no `PpState`.** This is the
  levelization constraint that put `PpStyle` in xo-ppsink in the first place
  (`xo-deps --why=xo-indentlog2:xo-ppsink -q` → `xo-indentlog2 -> xo-ppsink`).
  So `push_style`/`pop_style` must be expressible on **`PpSink`**, with
  `FlatSink` applying them immediately to its own style and `PrettySink`
  emitting tokens. Do not build a mechanism only PrettySink can use.

- **What does a freshly built sink start from?** A stack fixes style *during* a
  render; it says nothing about the initial value, which is still copied from a
  process-wide default. Whatever answers that has to survive: a program does
  want to say "tags are orange" once.

- **Colour is currently materialised at emission, not layout.** `color_guard`
  writes escape text into `k_string` tokens as they are emitted, which is why
  `count_visible_chars` has to skip them. If style becomes a token consumed at
  layout time, colour *could* move there too — enabling decisions that depend on
  how the group laid out. That is a bigger change and should be an explicit
  choice, not a side effect.

## Two concerns that are separable

1. **Thread safety** of the default and its guards.
2. **Scoped style with a defined extent** during a render.

(2) subsumes (1) for anything rendering through a sink, but not for the initial
value in (3) above. If (2) is deferred, the cheap interim for (1) is to make
`default_style()` return a **`thread_local`** instance — one word, removes the
race, and changes the meaning: a style set in `main()` no longer reaches worker
threads. That tradeoff is the reason it is worth stating rather than just doing.

## Four tickets now circle the same gap

- this one — scoped style, thread safety
- `.xo-backlog/xo-ppsink/issues/07` — nested formatting context (precision, base,
  date format per subtree)
- `.xo-backlog/xo-indentlog2/issues/02` — `PrettyContext`: configuration as a value
- `.xo-backlog/xo-indentlog2/issues/03` — bounded rendering (max-items/max-depth),
  also inherently scoped

Issue 07 already says these should be designed together or sequenced
deliberately, because four mechanisms for "richer sink configuration" would be
worse than one. **This ticket is the one with a concrete proposal**, so it is
the natural place to start.

**Files:**
- `xo-ppsink/include/xo/ppsink/PpStyle.hpp` — `default_style()`,
  `default_style_guard`
- `xo-ppsink/include/xo/ppsink/PpSink.hpp` — where push/pop must be expressible
- `xo-indentlog2/include/xo/indentlog2/print/PpState.hpp` — the proposed home
  for the stack
- `xo-indentlog2/include/xo/indentlog2/print/PpTokenType.hpp` — the free type
  values
- `xo-ppsink/src/ppsink/LogState.cpp:38` — the `thread_local` sink that makes
  the global inconsistent

**Done when:**
- concurrent renders on different threads cannot race on style, and there is a
  test that would have caught it
- `default_style_guard` is either gone or safe (`color_enabled_guard`, the
  other instance, was resolved on 2026-08-09 by folding its gate into
  `PpStyle` — the same shape of answer is available here)
- the relationship to issues 07, `xo-indentlog2/02` and `xo-indentlog2/03` is
  stated, not left implicit

## Notes

`default_style_guard` was added the same day as the defect, to rebase ~60
assertions onto the new colour defaults (`.xo-backlog/xo-printable2/issues/01`).
It is currently used only by single-threaded unit tests, which is why nothing
has failed — the tests are the only caller, and they never run two renders at
once. That is an argument about exposure, not about correctness.

Note the asymmetry with `color_enabled_guard`, retired the same day: that one
could be dissolved because its gate had a natural per-sink home and every
consumer already held a sink. `default_style()` is harder precisely because its
job is to supply the value a sink starts with, before any sink exists — which
is why a stack during a render does not, on its own, finish this ticket.
