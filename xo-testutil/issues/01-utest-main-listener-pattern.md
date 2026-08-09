# 01 — utest mains: replace the static-initializer trick with an explicit `main()`

Status: six of seven converted 2026-08-09; open only on the xo-ppsink decision
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
utest CMakeLists, as `xo-alloc2/utest/CMakeLists.txt` does.

### Answered 2026-08-09: the utest-only dependency IS a subsystem edge

It is not free. `xo_dependency()` records an edge for
`xo_emit_dependency_edges()` regardless of whether the target is a library or a
test executable (`xo-cmake/cmake/xo_macros/xo_cxx.cmake:1571`), so the edge
appears in the build-generated list. That is why the checked-in file already
carries xo-alloc2's:

```bash
grep -n '^xo-testutil ' xo-cmake/etc/xo/subsystem-edges
#   174:xo-testutil xo-alloc2
#   175:xo-testutil xo-indentlog2
```

It is nevertheless safe, because xo-testutil is close to the bottom of the
graph and none of the six is upstream of it:

```bash
xo-deps --deps-of=xo-testutil --format=names -q
#   xo-ppsink  xo-subsys  xo-testutil  xo-timeutil
for s in xo-ratio xo-reflect xo-webutil; do xo-deps --why=xo-testutil:$s -q || echo "no path: $s"; done
#   no path: xo-ratio / xo-reflect / xo-webutil
```

No cycle is possible for any of the six. **This does not generalise** — a
subsystem that xo-testutil itself depends on (xo-ppsink, xo-subsys, xo-timeutil)
cannot take this dependency, which is the same constraint that makes xo-ppsink
the odd one out below.

**Consequence: `xo-cmake/etc/xo/subsystem-edges` now lags the generated list.**
The checked-in copy is published by hand, and neither 525b1ca3 nor the
2026-08-09 conversions re-published it:

```bash
diff <(sort .build/subsystem-edges) <(sort xo-cmake/etc/xo/subsystem-edges)
#   only-in-generated: xo-testutil {xo-alloc, xo-expression, xo-interpreter,
#                                   xo-ratio, xo-reflect, xo-webutil,
#                                   xo-object2, xo-stringtable2}
#                      xo-indentlog2 {xo-gc, xo-object, xo-object2, xo-stringtable2}
```

Republish with `./reconfigure --capture-subsystem-edges` (the message
`xo_emit_dependency_edges: complete` in the configure output is the signal that
the capture would be whole rather than partial). Not done here: it rewrites
edges unrelated to this ticket, so it wants to be its own commit.

## Done 2026-08-09: six of seven

| subsystem | commit |
|---|---|
| xo-alloc | `525b1ca3` xo-alloc: use xo-testutil utest setup [REFACTOR] |
| xo-expression, xo-interpreter, xo-ratio, xo-reflect, xo-webutil | uncommitted at time of writing |

Each is the `xo-indentlog2/utest/indentlog2_utest_main.cpp` shape verbatim —
`CATCH_CONFIG_EXTERNAL_INTERFACES`, `CATCH_REGISTER_LISTENER(UtestListener)`,
own `main()` with `PpStyle::default_style() = PpStyle::plain()` as its first
statement — plus one `xo_dependency(<utest-exe> xo_testutil)` line each.

Verified, not assumed. An empty catch2 registry is the exact failure this
pattern risks (see the `CATCH_CONFIG_RUNNER` note above), and it exits 0 —
another instance of absence looking like success — so the test counts were read
rather than the sweep's `ok` taken at face value:

```bash
xo-build -q -k --utest xo-expression xo-interpreter xo-ratio xo-reflect xo-webutil
#   all five ok
for s in expression interpreter ratio reflect webutil; do $(find xo-$s/.build -name utest.$s -type f) | tail -2 | head -1; done
#   42 assertions in 6 test cases    (expression)
#   46 assertions in 3 test cases    (interpreter)
#   4099 assertions in 8 test cases  (ratio)
#   172 assertions in 15 test cases  (reflect)
#   12 assertions in 6 test cases    (webutil)
xo-reflect/.build/utest/utest.reflect --announce | head -1
#   Starting unit test: [struct-reflect-empty] at [.../StructReflector.test.cpp:24]
```

The `--announce` line is the check that the *listener* registered, which is new
behaviour these binaries did not have before; the assertion counts are the check
that the registry is not empty. That the rendering tests (`pretty.test.cpp`,
`ratio_pp.test.cpp`, `TypeDescr_pp.test.cpp`, `EndpointDescr.test.cpp`) still
pass is the check that `plain()` is still installed early enough.

Full sweep unchanged: 61 subsystems, 34 ok / 26 no-tests / 1 failed (xo-jit).

**Loose end from 525b1ca3:** `xo-alloc/utest/alloc_utest_main.cpp` opens
`@file indentlog2_utest_main.cpp` — copy-paste from the file it was modelled on.
One line, wrong in a way `grep` for a filename will believe.

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
- ~~the `Progress:` count is 0 for the six~~ — done 2026-08-09; the count is
  now **1**, and the one left is xo-ppsink
- xo-ppsink has an explicit decision recorded here, whichever way it goes.
  Now the *only* thing standing between this ticket and closed. The evidence
  above sharpens it: a utest-only `xo_dependency` is a real edge, and xo-ppsink
  is upstream of xo-testutil, so adopting the pattern there would close a cycle
  in the generated graph — not merely look like one. Either xo-ppsink keeps the
  initializer with a comment saying why, or the style-setting moves somewhere
  that does not need xo-testutil.
- the full sweep is unchanged: `xo-build -q -k --utest $SUBS` still gives
  34 ok / 26 no-tests / 1 failed (xo-jit)

## Notes

The defaults themselves are asserted in exactly one place,
`xo-ppsink/utest/PpStyle.test.cpp`, precisely because every other binary
installs `plain()`. Whatever replaces the initializer must not disturb that
file, or the real defaults become untested again — which is how ppsink's colour
gate sat at `false` unnoticed.
