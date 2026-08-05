# 01 — utest.jit: MachPipeline failures after restore to top-level build

Status: open
Type: bug

`utest.jit` is red. Two distinct failures in
`xo-jit/utest/MachPipeline.test.cpp`, with different characters:

| Test | Line | Rate | Symptom |
|---|---|---|---|
| `machpipeline.fptr` | 114 | **6/6 — deterministic** | C++ exception escapes the test |
| `machpipeline.struct` | 283 | **~1/6 — intermittent** | `SIGSEGV` |

Both predate the xo-ppsink migration work: measured on 6 runs each at
`4d0aa321` and with the xo-distribution migration applied, the two
distributions are identical. Neither is a regression from that work — xo-jit
includes no header it touched. They most likely arrived with
`c98860ea xo-jit: restore to top-level build + fix drift [BUGFIX]`
(2026-08-02), which re-enabled the target after a period out of the build.

Consequence today: a plain `ctest --test-dir .build` over the umbrella is red.
Everything else passes (32/32 with `-E utest.jit`).

**Still current as of 2026-08-05.** Full umbrella build + `ctest --test-dir .build`:
32 of 33 test binaries pass, `utest.jit` the sole failure, at
`MachPipeline.test.cpp:114` — 20 of 21 assertions pass. `machpipeline.struct`
(`:283`) did not fire in that run, which is consistent with its ~1-in-6 rate;
do not read a single green run as evidence it is fixed.

## 1. `machpipeline.fptr` — deterministic, and NOT a memory bug

This one is a logic error, not corruption. The thrown message:

```
:i_tc 1
LocalSymtab::assign_parent(P2): already have established parent P1
  :P1 <LocalSymtab :argv "[<Variable :name x2 :type double>]">
  :P2 <GlobalEnv :size 2>
```

So on the **second** testcase in the loop (`:i_tc 1`) a `LocalSymtab` that
already has a parent (an enclosing `LocalSymtab` binding `x2`) is asked to take
`GlobalEnv` as its parent instead, and `assign_parent` refuses.

Read that as environment/ownership state leaking across testcases, or a lambda
in function position re-parenting a symtab that was already installed. The
test's own docstring says the case relies on "lambda in function position" and
"argument with function type", which is exactly where a symtab would get
re-parented.

Start here — it reproduces every time, needs no sanitizer, and the failure
already names the two symtabs involved. Likely in `xo-expression`
(`LocalSymtab::assign_parent`) rather than in xo-jit itself.

## 2. `machpipeline.struct` — intermittent SIGSEGV

Fails roughly 1 run in 6 on identical binaries, so it is genuinely
nondeterministic rather than input-dependent. Candidates worth separating:

- uninitialized read (varies with stack/heap garbage)
- use-after-free of JIT'd code or of an LLVM ORC resource
- ASLR-sensitive addressing in the generated code

**Run it under a sanitizer** — the build already supports it, no new
plumbing needed (`xo-cmake/cmake/xo_macros/xo_cxx.cmake:771-792`):

```bash
cmake -S . -B .build-asan -DXO_ADDRESS_SANITIZE=ON
cmake --build .build-asan --target utest.jit -j6
./.build-asan/xo-jit/utest/utest.jit
```

`XO_ADDRESS_SANITIZE=ON` adds `-fsanitize=address` to both compile and link,
and swaps in `XO_ADDRESS_SANITIZE_COMPILE_OPTIONS` (which adds
`-Wno-macro-redefined` to dodge the `_FORTIFY_SOURCE` redefinition ASan
otherwise trips over).

Because it is ~1-in-6, loop it rather than running once:

```bash
for i in $(seq 20); do ./.build-asan/xo-jit/utest/utest.jit >/dev/null 2>&1 \
  || echo "failed on run $i"; done
```

Two caveats specific to this target:

- **LLVM ORC and ASan interact badly** unless LLVM itself is ASan-aware;
  expect false positives from JIT'd frames. If ASan is too noisy, UBSan is
  lighter and there is no `XO_UNDEFINED_SANITIZE` knob yet — adding one beside
  `XO_ADDRESS_SANITIZE` is a small change to the same block.
- The test constructs an RNG (`rng::Seed`, `xoshiro256ss`) with the seeding
  lines commented out. Confirm the seed is actually fixed; if any run-to-run
  variation enters there, that is a cheaper explanation than memory corruption
  and should be ruled out first.

## Files

- `xo-jit/utest/MachPipeline.test.cpp` — both failing cases (`:114`, `:283`)
- `xo-expression/include/xo/expression/LocalSymtab.hpp` and its `.cpp` —
  `assign_parent`, the source of the fptr diagnostic
- `xo-cmake/cmake/xo_macros/xo_cxx.cmake:771-792` — sanitizer options

## Done when

- `utest.jit` passes 20 consecutive runs
- `ctest --test-dir .build` is green with no `-E` exclusion
- if the struct failure turns out to be a real memory bug, an ASan run is
  clean as well as a plain one

## Notes

Do not "fix" this by disabling the tests. `machpipeline.struct` covers JIT
struct support and `machpipeline.fptr` covers function-typed arguments; both
are load-bearing for the interpreter path.
