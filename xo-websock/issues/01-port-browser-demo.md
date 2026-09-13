# 01 — xo-websock/utest/ is a pre-umbrella browser demo, not a unit test

Status: open
Type: port / misfiled
Raised: 2026-09-13, while checking whether this was another `xo-reader/issues/01`

`xo-websock/CMakeLists.txt:23` carried a bare `#add_subdirectory(utest)` with no
reason given. It looked like `xo-reader`'s disabled suite. **It is not the same
thing, and the difference matters:** xo-reader had ~120 passing assertions going
dark because one case failed. Here there is no test at all.

## What it actually is

A **manual browser demo**. `utest.websock` serves http on port 7682; you point a
browser at `localhost:7682/ex_websock.html` (`xo-websock/utest/README`). Its own
CMakeLists says so and leaves the registration out:

```cmake
## note: can't add this yet,  because test not automated.
##       requires manual interaction from browser
#add_test(NAME ${SELF_EXE} COMMAND ${SELF_EXE})
```

So enabling `add_subdirectory(utest)` would register **zero** ctest tests even
if it compiled. It is misfiled: `utest/` is the wrong directory for it.

It is a websocket + d3 view of a live Kalman filter — `mount-origin/` still
holds `ex_websock.html`, `ex_websock.js`, `d3ex`, and the svg assets.

## Why it does not build

Measured 2026-09-13. It does not fail at configure — cmake turns the unknown
`option` and `volfit` link targets into plain `-l` flags — it fails at the first
compile:

```
websock_utest_main.cpp:3:10: fatal error: filter/KalmanFilterSvc.hpp:
                             No such file or directory
```

25 includes, all written before the `xo/` prefix convention. Resolve each
against today's tree:

```bash
grep -oP '#include "\K[^"]+' xo-websock/utest/websock_utest_main.cpp | sort \
| while read i; do b=${i#*/}
    f=$(ls xo-*/include/xo/*/$b 2>/dev/null | head -1)
    printf '%-34s %s\n' "$i" "${f:-MISSING}"; done
```

| | count | today |
|---|---|---|
| `process/`, `randomgen/`, `reactor/`, `simulator/`, `printjson/`, `websock/`, `indentlog/` | 13 | same name, gains `xo/` prefix |
| `filter/` | 2 | renamed → `xo/kalmanfilter/` (both files present) |
| `time/Time.hpp` | 1 | renamed → `xo/timeutil/timeutil.hpp` |
| `option/` | 5 | **does not exist in this tree** |
| `volfit/` | 3 | **does not exist in this tree** |

So **17 of 25 are pure renames** and 8 are not. Worth stating precisely: the
CMakeLists comment says only "Need to port option, volfit", which undersells the
include churn, and an earlier draft of this analysis said "three subsystems do
not exist (filter, option, volfit)" — wrong, `filter` is a rename to
`xo-kalmanfilter` and its two headers are both there.

The README dates it: `cd path/to/kalman/build/src/websock/utest` — from before
the repo was called xo.

## Why it might be worth porting rather than deleting

RC's stated longer-range goal behind the `reflectable2` milestone is **xo state
reachable from a browser, so animations are grounded in actual state rather than
a parallel model**. This is a working precedent for exactly that, from the
kalman era: a reactor-driven simulation streaming over a websocket into d3.

That is the whole argument for keeping it. Against: `option` and `volfit` were
never ported, and porting them is not a side quest — `option/` alone is
`OptionStrikeSet`, `PricingContext`, `StrikeSetMarketModel`, `StrikeSetOmd`,
`UlMarketModel`.

## Options

1. **Delete it.** Two subsystems missing, includes two migrations out of date.
   Cheapest, and loses a precedent for the browser goal.
2. **Move to `example/` and port the 17 renames**, stubbing or dropping the
   parts that need `option`/`volfit`. Unverified whether the demo is meaningful
   without the volatility-fit content — that is the first thing to find out, and
   it decides whether this option exists.
3. **Move to `example/` and port `option`/`volfit` too.** Largest, and only
   justified if those subsystems are wanted for their own sake.

Not chosen here. What was done 2026-09-13: the bare `#add_subdirectory(utest)`
was replaced with a comment recording all of the above, so the next person does
not re-derive it. That is the whole change.

## Done when

- [ ] a decision among 1/2/3 is recorded here
- [ ] if 2 or 3: the demo lives under `example/`, not `utest/`, and builds
- [ ] `xo-websock/CMakeLists.txt` no longer carries a commented-out
      `add_subdirectory` — either the directory is gone or it is built

## Notes

`xo-websock` reports `ok (utest:no-tests)` in `xo-build --sweep` and always has.
Nothing about that line distinguishes "no tests written" from "a test directory
exists but is commented out of the build" — which is how this sat unexamined.
The same was true of `xo-reader` until 2026-09-13, and there it was hiding ~120
passing assertions.

The bare grep is too noisy to use — 25 subsystems carry a commented-out
`add_subdirectory(utest)`, almost all of it scaffold boilerplate with no
`utest/` directory behind it. What discriminates is **commented out AND test
source present**:

```bash
for f in $(grep -rln '^\s*#\s*add_subdirectory(utest)' xo-*/CMakeLists.txt); do
    s=$(dirname $f)
    n=$(ls $s/utest/*.cpp $s/utest/*.py 2>/dev/null | wc -l)
    [ "$n" -gt 0 ] && printf '%s (%d files)\n' $s $n
done
# xo-websock (1 files)     <- 2026-09-13, after xo-reader was restored
```

That found one false positive worth fixing rather than working around:
`xo-pyobject2/CMakeLists.txt` had a live `add_subdirectory(utest)` on line 22
*and* a leftover scaffold `#add_subdirectory(utest)` on line 23. Its tests run
fine; the comment was vestigial. Removed, so the check above means what it says.

Progress: `for f in $(grep -rln '^[[:space:]]*#[[:space:]]*add_subdirectory(utest)' xo-*/CMakeLists.txt); do s=$(dirname $f); n=$(ls $s/utest/*.cpp $s/utest/*.py 2>/dev/null | wc -l); [ "$n" -gt 0 ] && echo $s; done | wc -l`
