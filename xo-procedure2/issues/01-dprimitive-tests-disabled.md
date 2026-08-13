# 01 — 5 of 6 DPrimitive tests are compiled out behind `#ifdef OBSOLETE`

Status: diagnosed
Type: test-gap
Progress: sed -n '/#ifdef OBSOLETE/,/#endif/p' xo-procedure2/utest/DPrimitive.test.cpp | grep -c TEST_CASE

`xo-procedure2/utest/DPrimitive.test.cpp` contains six `TEST_CASE`s. **One
compiles.** The other five sit inside an `#ifdef OBSOLETE` block and have never
run in any build here — `OBSOLETE` is not defined anywhere:

```bash
grep -rn 'OBSOLETE' --include=CMakeLists.txt --include=*.cmake xo-procedure2/ xo-cmake/ | grep -v '/\.build/'
# (no output)

grep -n '#ifdef OBSOLETE\|#endif\|TEST_CASE' xo-procedure2/utest/DPrimitive.test.cpp
```

Live: `DPrimitive-init`.

Disabled:

| test | what it would check |
|---|---|
| `DPrimitive-n_args` | `s_mul_gco_gco_pm.n_args() == 2` |
| `DPrimitive-is_nary` | `s_mul_gco_gco_pm.is_nary() == false` |
| `DPrimitive-apply_nocheck-float-float` | primitive application, both args float |
| `DPrimitive-apply_nocheck-int-int` | ditto, ints; asserts the result is a `DInteger` of 21 |
| `DPrimitive-apply_nocheck-int-float` | ditto, mixed — i.e. the numeric-dispatch path |

That is the whole behavioural coverage of primitive *application* in this
subsystem, and none of it runs.

Found 2026-08-12 while deleting a sixth disabled case, `DPrimitive-pretty`,
during phase E of `.xo-backlog/xo-printable2/issues/01-aprintable-pretty-ppsink.md`.
That one was genuinely obsolete — it tested the deprecated print protocol and is
superseded by `DPrimitive-render` — so it was removed rather than re-enabled.
The other five are a different question and are not print-related at all.

## Why it was invisible

The suite reports success. `xo-build --sweep` counts xo-procedure2 among the
passing subsystems, and nothing distinguishes "6 tests pass" from "1 test passes
and 5 do not exist". This is the same failure mode as the disabled-tests sweep
recorded for the nix packages — a suite that runs is not a suite that covers.

It also survived a text search: the list of files touching the deprecated print
protocol was built with `grep -rln ppstate_standalone`, which matches inside
`#ifdef` blocks, so this file was wrongly counted as an active consumer. Kept
here per rule 6 — **a grep over source text cannot tell you what compiles.**

## What to do

Unknown, and that is the point: nobody has established **why** they were
disabled. `git log -S'ifdef OBSOLETE' -- xo-procedure2/utest/DPrimitive.test.cpp`
attributes it to the subrepo clone commit (`e2d181dd`), so the decision predates
this tree's history and its reason did not travel.

Before re-enabling, check whether they still reflect the API:

- `Primitives::s_mul_gco_gco_pm` — does that symbol still exist with that
  spelling? The type has since been renamed `DPrimitive_gco_2_gco_gco`.
- `DSimpleRcx` / `ARuntimeContext` construction in the bodies — the runtime
  context has moved subsystems at least once.
- `DArray::array(alloc, x, y)` — variadic form still present?

If they compile and pass, delete the `#ifdef`. If they do not, they are stale
scaffolding and should be deleted outright rather than left looking like
coverage — but that is a decision to make **after** compiling them, not from
reading.

Either outcome ends with this file having no `#ifdef OBSOLETE` in it, which is
what the `Progress:` count measures.
