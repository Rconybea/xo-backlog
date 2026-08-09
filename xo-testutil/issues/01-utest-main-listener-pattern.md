# 01 — utest mains: replace the static-initializer trick with an explicit `main()`

Status: open
Type: refactor
Progress: grep -rl 's_plain_pp_style' --include=*utest_main*.cpp xo-*/ | grep -v '/\.build/' | wc -l

Raised by RC 2026-08-09, after asking how `s_plain_pp_style` in
`xo-alloc/utest/alloc_utest_main.cpp` gets to run. That the question needed
asking is the argument for this ticket.

## What is there now

Seven utest mains set the ppsink presentation style by smuggling a side effect
into a namespace-scope initializer:

```cpp
namespace {
    const bool s_plain_pp_style = []() {
        xo::pp::PpStyle::default_style() = xo::pp::PpStyle::plain();
        return true;
    }();
}
```

```bash
grep -rl 's_plain_pp_style' --include=*utest_main*.cpp xo-*/ | grep -v '/\.build/'
#   xo-alloc  xo-expression  xo-interpreter  xo-ppsink
#   xo-ratio  xo-reflect     xo-webutil
#   (re-run rather than trusting this list)
```

It works, and the ordering is sound: the variable's initializer is a lambda
*call*, hence dynamic initialization, which runs before `main()` — and `main()`
is generated into the same translation unit by `CATCH_CONFIG_MAIN`, so the
standard's allowance for deferring dynamic initialization still sequences it
first.

It is there **only because `main()` is not ours to edit.** `CATCH_CONFIG_MAIN`
generates it. The `const bool` is scaffolding: nothing ever reads it.

Why it was written this way in the first place: it is one line per subsystem
against ~60 assertions across 8 subsystems that would otherwise each need a
guard. See `.xo-backlog/xo-printable2/issues/01`, the PpStyle section. That
tradeoff still holds — this ticket changes *where* the declaration lives, not
whether it is centralised.

## Why change it

- **Obscure.** It reads as an unused variable. A reader has to know the idiom
  to see that a statement is being executed at all.
- **Action at a distance, undocumented at the far end.** A reader of
  `xo-alloc/utest/GcStatistics.test.cpp` sees colourless expectations and
  nothing nearby explaining why.
- **A real, if not-yet-live, hazard.** A `PpSink`, `PpConfig` or `PpStyle`
  constructed as a *namespace-scope global in another TU* has unspecified
  ordering against this initializer, and could capture the legacy defaults —
  failing as a confusing colour mismatch in one file. Checked 2026-08-09: every
  sink and config in the tree's tests is inside a function, so nothing is
  exposed today.

```bash
# the check, worth re-running if a lone rendering test ever fails
grep -rn '^\s*\(static \)\?\(const \)\?\(PpConfig\|FlatSink\|PrettySink\|PpStyle\)[ &]' \
     --include=*.test.cpp xo-*/utest/ | grep -v '/\.build/'
```

An explicit `main()` has none of this: the statement is a statement, it runs
where a reader expects, and its order relative to everything else is obvious.

## The pattern to copy: `xo-alloc2/utest/alloc2_utest_main.cpp`

Own `main()`, `UtestAppStart`, and a registered listener:

```cpp
#define CATCH_CONFIG_EXTERNAL_INTERFACES   // before UtestListener.hpp

#include <xo/testutil/UtestAppStart.hpp>
#include <xo/testutil/UtestListener.hpp>

namespace xo { CATCH_REGISTER_LISTENER(UtestListener); }

int main(int argc, char * argv[]) {
    auto app = xo::UtestAppStart("utest.alloc2");
    int retval = app.init(argc, argv);
    if (retval) return retval;
    /* <-- PpStyle::default_style() = PpStyle::plain(); goes here, visibly */
    app.setup();
    return app.run();
}
```

**Do not reach for `CATCH_CONFIG_RUNNER`.** That file already carries the
reason, and it is not obvious: the catch2 implementation is compiled once, into
`libxo_testutil` (`UtestAppStart.cpp`). A second copy in the executable gets its
own test registry; linux ELF interposition merges them and hides it, macOS's
two-level namespace does not, and the runner in the dylib sees an empty registry
→ "No tests ran". `CATCH_CONFIG_EXTERNAL_INTERFACES` pulls in just the
listener interfaces. Keep that comment when copying.

`xo-indentlog2/utest/indentlog2_utest_main.cpp` is the smaller variant — same
shape, no `ThreadPrettySink` install — and already sets the style as a plain
statement. It is the closest thing to a target for the seven.

## The part that is NOT uniform: xo-ppsink cannot do this

`UtestAppStart` lives in xo-testutil, and **xo-testutil depends on xo-ppsink**:

```bash
xo-deps --why=xo-testutil:xo-ppsink -q
#   xo-testutil -> xo-ppsink
grep -n 'xo-ppsink xo-testutil' xo-cmake/etc/xo/subsystem-edges
#   84:xo-ppsink xo-testutil
```

So `xo-ppsink`'s own utest adopting the pattern points a test target at a
subsystem that depends on the one under test. That may be tolerable at link time
— a utest executable is not the library — but it is a cycle in the declared
subsystem graph, and nix builds each package from its declared inputs, so it
should not be waved through. **Decide xo-ppsink separately**; the other six are
uniform.

Where the other six stand, measured 2026-08-09:

```bash
for s in xo-alloc xo-expression xo-interpreter xo-ratio xo-reflect xo-webutil; do
    printf '%-16s ' $s; xo-deps --why=$s:xo-testutil -q || echo "NO PATH"
done
#   xo-alloc         xo-alloc -> xo-indentlog2 -> xo-testutil
#   xo-expression    xo-expression -> xo-indentlog2 -> xo-testutil
#   xo-interpreter   xo-interpreter -> xo-indentlog2 -> xo-testutil
#   xo-ratio         NO PATH
#   xo-reflect       NO PATH
#   xo-webutil       NO PATH
```

Three already reach xo-testutil transitively. Three do not and would gain a
**test-only** dependency — `xo_dependency(${UTEST_EXE} xo_testutil)` in the
utest CMakeLists, as `xo-alloc2/utest/CMakeLists.txt` does. Worth confirming
that a utest-only dependency does not have to be added to `subsystem-edges`
(xo-alloc2 links xo_testutil in its utest; check whether its edge list says so)
before assuming it is free.

## Scope note

Only the seven carry the initializer. Of the other utest mains, ~20 use bare
`CATCH_CONFIG_MAIN` and set no style at all — they render nothing and need
nothing. **Do not convert them as part of this.** The value here is removing an
obscure idiom where it exists, not uniformity for its own sake.

**Files:**
- `xo-alloc2/utest/alloc2_utest_main.cpp` — the pattern, with the
  `CATCH_CONFIG_EXTERNAL_INTERFACES` rationale
- `xo-indentlog2/utest/indentlog2_utest_main.cpp` — the smaller variant,
  already setting the style as a statement
- `xo-alloc2/utest/CMakeLists.txt` — how the test-only xo_testutil dependency
  is declared
- the seven `*_utest_main.cpp` from the grep above
- `xo-ppsink/include/xo/ppsink/PpStyle.hpp` — what is being set, and why tests
  set it

**Done when:**
- the `Progress:` count is 0 for the six, each setting the style as a statement
  in its own `main()`
- xo-ppsink has an explicit decision recorded here, whichever way it goes
- the full sweep is unchanged: `xo-build -q -k --utest $SUBS` still gives
  34 ok / 26 no-tests / 1 failed (xo-jit)

## Notes

The defaults themselves are asserted in exactly one place,
`xo-ppsink/utest/PpStyle.test.cpp`, precisely because every other binary
installs `plain()`. Whatever replaces the initializer must not disturb that
file, or the real defaults become untested again — which is how ppsink's colour
gate sat at `false` unnoticed.
