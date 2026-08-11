# 05 — `toppstr(x)` with no config silently stopped colouring

Status: fixed 2026-08-09
Type: bug (output-visible)
Milestone: ppsink-migration

## Resolution (RC, 2026-08-09)

Reading 1 below — **regression**. The no-config overload now uses
`PpConfig::colored()`:

```cpp
// xo-indentlog2/include/xo/indentlog2/print/toppstr.hpp:60
return toppstr(PpConfig::colored(), a0, args...);
```

Re-probed against the installed headers: `toppstr(tag("k", x))` is
`"\033[38;5;245m:k\033[0m 1"` again, matching both `toppstr(PpConfig(), ...)`
and `toppstr(PpConfig::colored(), ...)`.

Pinned by **`toppstr-no-config-is-colored`**
(`xo-indentlog2/utest/toppstr.test.cpp`), which is the part that was missing —
the overload picks its own config, so no other test in the file constrains what
style that is. Two assertions on observed literals, two asserting it is
`PpConfig::colored()` specifically. Mutation-checked: switching the overload
back to `plain()` fails exactly one assertion out of 583.

Deliberately independent of `PpStyle::default_style()`, which this binary's
`main()` replaces with `plain()`; `PpConfig::colored()` names its colours.

Full sweep after the change: 61 subsystems, 34 ok / 26 no-tests / 1 failed
(`xo-jit machpipeline.fptr`, pre-existing) — so no test elsewhere was relying
on a bare `toppstr(x)` being colourless.

(Those are the 2026-08-09 totals. The tree reads 62 / 34 / 28 / **0** as of
2026-08-11 — two placeholder subsystems scaffolded, and `utest.jit` fixed rather
than permanent. The conclusion above is unaffected; only the numbers moved.)

---


`522799d7` ("xo-indentlog2: streamline scratch PpSink/PpConfig") changed the
no-config `toppstr` overload from

```cpp
return toppstr(PpConfig(), a0, args...);          // before
return toppstr(PpConfig::plain(), a0, args...);   // after
```

`PpConfig()`'s `PpStyle` member is default-constructed, i.e. **coloured**
(`PpStyle::color_enabled = true`, `PpStyle.hpp:31`). `PpConfig::plain()` is
`PpStyle::plain()`. So `toppstr(x)` no longer emits colour, while
`toppstr(cfg, x)` for any coloured `cfg` still does.

## Measured

Compiled against the installed headers after `522799d7`:

```cpp
int x = 1;
toppstr(tag("k", x));                        // ":k 1"                    <-- was coloured
toppstr(PpConfig().with_logbuf_size(65536)
                  .with_logbuf_name("probe"),
        tag("k", x));                        // "\e[38;5;245m:k\e[0m 1"
toppstr(PpConfig::colored(), tag("k", x));   // "\e[38;5;245m:k\e[0m 1"
```

(The middle case needs the explicit size and name because a bare `PpConfig()`
has a zero-sized, unnamed logbuf — see issue 01.)

## Why it matters

This is precisely the regression that `.xo-backlog/xo-printable2/issues/01`'s
"Colour: ppsink's gate now defaults ON" section was written to prevent: RC's
call there was that ppsink must not be the reason a call site loses colour when
it migrates off legacy `toppstr2`, because that turns phase D into a
colour-restoring exercise at every site.

## Why no test caught it

Every test in `xo-indentlog2/utest/toppstr.test.cpp` renders deliberately
without colour, in order to pin text. The one test guarding a colour default,
`color-enabled-by-default`, guards `xo::pp::color_config::color_enabled` — a
different thing from `PpConfig`'s `PpStyle` member, which is what the overload
selects.

`522799d7` also rewrote `toppstr-overloads` from
`toppstr(PpConfig(), 123)` to `toppstr(PpConfig::plain(), 123)`, so the only
assertion that exercised the old default no longer does.

## Decision needed (RC) — decided, see Resolution above

Two defensible readings, and the ticket could not pick:

1. **Regression** — `toppstr(x)` should colour, matching `toppstr(cfg, x)` and
   legacy `toppstr2`. Fix: give the no-config overload a `colored()`-styled
   config, i.e. `PpConfig::colored()` or a `PpConfig::anon()` that keeps the
   default style and only supplies size + name.
2. **Intended** — a bare `toppstr(x)` is the "give me a plain string" entry
   point, and colour is opt-in via a config. Then the printable2 decision
   narrows to *configured* renders and that section should say so.

Either way it wants a test that fails if the answer changes, since none exists.

**Files:**
- `xo-indentlog2/include/xo/indentlog2/print/toppstr.hpp:60` — the overload
- `xo-indentlog2/src/indentlog2/PpConfig.cpp:113,120` — `plain()` / `colored()`
- `xo-indentlog2/utest/toppstr.test.cpp` — where the missing test goes
- `.xo-backlog/xo-printable2/issues/01` — the decision this bears on

**Done when:** ~~the behaviour of `toppstr(x)` is decided, pinned by a test, and
the printable2 colour section agrees with it.~~ All three done 2026-08-09.

## Notes

The general lesson, worth carrying into the rest of the migration: **a default
that no test names is a default that can be flipped silently.** Both halves of
this happened within a day — `color_config::color_enabled` sat wrong-way-round
unnoticed for the same reason, and got `color-enabled-by-default` in response.
Any entry point that picks its own config rather than taking one needs a test
saying which config it picks.
