# 01 — Run the utest suite under AddressSanitizer in CI

Status: open
Type: task

Nothing in CI runs the tests under a sanitizer, so a memory-corruption bug in any
subsystem is invisible until it happens to segfault.

## Motivating incident (2026-08-08)

`utest.unit` segfaulted on forgejo under **both** gcc and clang, and passed
locally 5/5 in both the umbrella and standalone builds. The cause was a
stack-buffer overflow in `xo-flatstring`:

```cpp
/* flatstring::append(const flatstring<N2> &, pos, count) -- BEFORE */
std::size_t i_dest = size();
for (; i_src < std::min(..., N-1);      /* bound is on i_src ... */
     ++i_src, ++i_dest)
    value_[i_dest] = ...;               /* ... but i_dest reaches size() + N - 1 */
```

Appending onto an *empty* flatstring is safe, so the first append of any sequence
looks fine and every one after it corrupts. xo-unit tripped it through
`natural_unit::abbrev()`, which joins several `bpu` abbrevs into a
`flatstring<32>`. Fixed in `e8c87cb7`, tests in `5719f9f2`.

**A stack overflow corrupts silently far more often than it crashes** -- whether
it faults depends on stack layout, which is why this reproduced only on the CI
host. Running the same tests under ASan found it in one run, immediately, with an
exact stack. That is the argument for this ticket: the bug existed for a long
time and only luck decided where it surfaced.

## What exists already

The knob is in place -- no new plumbing needed
(`xo-cmake/cmake/xo_macros/xo_cxx.cmake`, ~line 771):

```cmake
if(XO_ADDRESS_SANITIZE)
    add_compile_options(-fsanitize=address)
    add_link_options(-fsanitize=address)
endif()
```

It also swaps in `XO_ADDRESS_SANITIZE_COMPILE_OPTIONS`, which adds
`-Wno-macro-redefined` to dodge the `_FORTIFY_SOURCE` redefinition ASan
otherwise trips over. Verified working by hand:

```bash
cmake -S xo-unit -B .build-asan -DENABLE_TESTING=1 -DXO_ADDRESS_SANITIZE=ON \
      -DCMAKE_MODULE_PATH=$PREFIX/share/cmake -DCMAKE_PREFIX_PATH=$PREFIX
cmake --build .build-asan -j6
./.build-asan/utest/utest.unit
```

`xo-build` has no `--sanitize` flag; it would need one, or the ASan job drives
cmake directly.

## Shape to decide

`.forgejo/workflows/ci-cmake.yaml` already runs a 2-way matrix (gcc, clang) with
`max-parallel: 1` because the host has 8GB. ASan roughly doubles build time and
memory, so **do not just add a third matrix leg to the existing job**.

Options:

- **(a) A separate workflow on a schedule** (nightly/weekly), one compiler only,
  building the whole chain with `XO_ADDRESS_SANITIZE=ON`. Cheapest to run, and
  the incident above would have been caught within a day.
- **(b) A separate workflow triggered manually / on demand** -- no recurring
  cost, but only catches what someone thinks to look for.
- **(c) Per-commit, gcc only.** Highest signal, highest cost; likely too slow for
  an 8GB host given the chain is ~40 subsystems.

**(a) is the recommendation**: this class of bug is latent for months, so nightly
is enough, and it keeps the per-commit pipeline's runtime unchanged.

## Watch out for

- **`utest.jit` is expected to fail** regardless -- see
  `.xo-backlog/xo-jit/issues/01`. It also mixes LLVM ORC with ASan, which
  interact badly unless LLVM itself is ASan-aware; expect false positives from
  JIT'd frames. Exclude it (`ctest -E utest.jit`) rather than let it mask
  everything else.
- **xo-tokenizer's utest is disabled** (`.xo-backlog/xo-tokenizer/issues/01` is
  resolved, but confirm it is in the build) -- a sanitizer job only covers tests
  that actually run.
- Some subsystems' tests are slow; check total wall-clock before choosing a
  cadence.
- There is **no `XO_UNDEFINED_SANITIZE` knob**. UBSan is cheaper than ASan and
  catches a different class (signed overflow, bad shifts, misaligned access).
  Adding one beside the ASan block is a small change and worth considering in the
  same pass.

**Files:**
- `.forgejo/workflows/ci-cmake.yaml` — the existing pipeline (do not slow it down)
- `xo-cmake/cmake/xo_macros/xo_cxx.cmake` — `XO_ADDRESS_SANITIZE`, and where a
  UBSan knob would go
- `xo-cmake/bin/xo-build.in` — where a `--sanitize` flag would go

**Done when:**
- the utest suite runs under ASan on some cadence, with results visible
- `utest.jit` is excluded, with a comment saying why
- a decision is recorded on UBSan

## Notes

Worth doing a **one-off full-tree ASan run before wiring up the schedule** --
this bug had been latent long enough that others are plausibly sitting in the
same state, and finding them in a batch is easier than one CI failure at a time.
