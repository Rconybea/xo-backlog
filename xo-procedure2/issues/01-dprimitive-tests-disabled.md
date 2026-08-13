# 01 — 5 disabled DPrimitive tests: restore them, in xo-numeric/ not here

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
| `DPrimitive-n_args` | `n_args() == 2` |
| `DPrimitive-is_nary` | `is_nary() == false` |
| `DPrimitive-apply_nocheck-float-float` | primitive application, both args float |
| `DPrimitive-apply_nocheck-int-int` | ditto, ints; asserts the result is a `DInteger` of 21 |
| `DPrimitive-apply_nocheck-float-int` | ditto, mixed — i.e. the numeric-dispatch path |

That is the whole behavioural coverage of primitive *application*, and none of
it runs.

Found 2026-08-12 while deleting a sixth disabled case, `DPrimitive-pretty`,
during phase E of `.xo-backlog/xo-printable2/issues/01-aprintable-pretty-ppsink.md`.
That one was genuinely obsolete — it tested the deprecated print protocol and is
superseded by `DPrimitive-render` — so it was removed rather than re-enabled.
The other five are not print-related and **are wanted back** (RC, 2026-08-13).

## They do not belong in xo-procedure2

RC, 2026-08-13: restore them in **`xo-numeric/utest/`**.

The reason is visible in what they reference. Both symbols they use are gone
from xo-procedure2:

```bash
grep -oE '(Primitives|NumericPrimitives)::[a-z_0-9]+' xo-procedure2/utest/DPrimitive.test.cpp | sort -u
#   NumericPrimitives::s_mul_gco_gco
#   Primitives::s_mul_gco_gco_pm

grep -rn 's_mul_gco_gco_pm' --include=*.hpp --include=*.cpp xo-*/ | grep -v '/\.build/'
#   xo-procedure2/include/xo/procedure2/init_primitives.hpp:24 -- and that
#   declaration is ITSELF inside #ifdef OBSOLETE, annotated
#   "see xo-numeric/src/numeric/NumericPrimitives.cpp"
```

So the tests were disabled because the thing they tested **moved**, not because
the coverage stopped mattering. xo-procedure2 now owns the primitive
*machinery* (`DPrimitive_gco_2_gco_gco`, `PrimitiveRegistry`); xo-numeric owns
the arithmetic primitives themselves, which is what `apply_nocheck-int-float`
is really exercising — the numeric dispatch.

Direction of dependency is right for the move:

```bash
xo-deps --why=xo-numeric:xo-procedure2      # xo-numeric -> xo-procedure2, rc=0
```

so a test in xo-numeric/utest can use both; the reverse would not work.

## The complication: no globals, so the fixture must build the primitives

The old tests took a **static**: `Primitives::s_mul_gco_gco_pm`. There is no
such object now. Primitives are constructed on demand from an allocator and a
string table:

```bash
grep -n 'make_.*_pm' xo-numeric/include/xo/numeric/NumericPrimitives.hpp
#   static DPrimitive_gco_2_gco_gco * make_multiply_pm(obj<AAllocator> mm,
#                                                      StringTable * stbl);
#   ... make_divide_pm, make_add_pm, make_subtract_pm, make_cmpeq_pm, ...
```

`xo-numeric/src/numeric/SetupNumeric.cpp:89` (`register_primitives`) is the
worked example of the calling convention:

```cpp
obj<AAllocator> mm = rcx.allocator();
StringTable * stbl = rcx.stringtable();

PrimitiveRegistry::install_aux(sink,
                               NumericPrimitives::make_multiply_pm(mm, stbl),
                               flags & InstallFlags::f_essential);
```

A test fixture needs the `mm` + `stbl` pair and can call the factory directly —
it does not need the registry or the install sink, which are about publishing
primitives into an environment, not about applying one.

Two things to work out when doing it, both **unverified**:

- `xo-numeric/utest/Numeric.test.cpp` already has a `Fixture` holding an
  `aux_arena_` (`:28`). Whether it also has a `StringTable`, or needs one
  added, has not been checked.
- the old tests built an `ARuntimeContext` via `DSimpleRcx` to pass to
  `apply_nocheck`. Whether that is still the right way to obtain one from a
  test — as opposed to going through a `DVsmRcx` or the numeric fixture — has
  not been checked either.

## Do not simply re-enable in place

Deleting the `#ifdef OBSOLETE` in `xo-procedure2/utest/DPrimitive.test.cpp`
will not compile: the statics are gone. The work is a **port**, not an
un-comment — rewrite the five against `NumericPrimitives::make_*_pm` in
xo-numeric/utest, then delete the dead block here.

The `Progress:` count reaching 0 therefore means "the dead block is gone from
xo-procedure2", which is the correct end state whether the tests are ported or
abandoned. It does **not** by itself mean the coverage exists — that is
`xo-numeric/utest` having the five cases, and is worth checking by name.
