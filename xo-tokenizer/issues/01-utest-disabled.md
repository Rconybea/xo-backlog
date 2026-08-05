# 01 — xo-tokenizer/utest is disabled; one real bug is keeping it out

Status: open
Type: bug

`xo-tokenizer/CMakeLists.txt:23` reads

```cmake
#add_subdirectory(utest)   # tests failing, temporarily remove
```

so `utest.tokenizer` is not built, not registered with ctest, and not run by
either the umbrella build or `.forgejo/workflows/ci-cmake.yaml`. The comment
predates the umbrella: the line arrived with the subrepo import
(`1f981a06`, 2026-06-06), so no in-tree history explains what was failing.

**It is one failing test case, not a broken suite.** Built and run by hand
(2026-08-05): **402 assertions, 401 pass**; 5 of 6 test cases pass. The single
failure is `TEST_CASE("tokenizer")` at `xo-tokenizer/utest/tokenizer.test.cpp:215`.

## The failure

```
input_state.hpp:250: void xo::scm::input_state<CharT>::advance_until(const CharT*)
  [with CharT = char]:
  Assertion `current_line_.lo() <= pos && pos <= current_line_.hi()' failed.
```

`advance_until()` is handed a `pos` outside the current line's span.

**The assertion is correct — it is catching a real bug, not being over-strict.**
Rebuilt with `-DNDEBUG` to mask it, the run gets further and then dies:

```
tokenizer.test.cpp:385: FAILED:
  SIGSEGV - Segmentation violation signal
```

So the out-of-range `pos` is subsequently dereferenced. Disabling or loosening
the assert is not a fix; it converts an abort into memory corruption.

The failing case loops over `s_testcase_v` (a table of input/expected-token
rows) and aborts partway through, so the first task is to identify *which*
input triggers it — the `i_tc` index is logged via `xtag("i_tc", i_tc)` but
only when `rehearser::enable_debug()` is on, and the abort kills the process
before the rehearser's debug re-run.

## Cost of leaving it

- The tokenizer's 400+ assertions never run in CI.
- **`xo-tokenizer/utest/span.test.cpp` (added 2026-08-05) is registered in
  `xo-tokenizer/utest/CMakeLists.txt` but never compiled.** It pins the
  `std::ranges::contiguous_range` contract that `xo::pp::hex_view` relies on
  structurally. It was verified by a manual compile+run at the time it landed;
  nothing verifies it now.

## Reproduce

The utest dir is out of the build, so build it by hand:

```bash
cd /home/roland/proj/xo-umbrella2
cmake --build .build --target xo_tokenizer -j6
CATCH=$(dirname $(dirname $(find ~/nixroot/nix/store -maxdepth 4 -name catch.hpp -path '*catch2*' | head -1)))
LIB=.build/xo-tokenizer/src/tokenizer/libxo_tokenizer.so.1
g++ -std=c++20 -I xo-tokenizer/include -I xo-indentlog/include \
    -I xo-timeutil/include -I xo-flatstring/include -I "$CATCH" \
    -o /tmp/utest.tokenizer \
    xo-tokenizer/utest/tokenizer_utest_main.cpp \
    xo-tokenizer/utest/span.test.cpp \
    xo-tokenizer/utest/token.test.cpp \
    xo-tokenizer/utest/tokenizer.test.cpp \
    "$LIB" -Wl,-rpath,$(dirname "$LIB")
/tmp/utest.tokenizer
```

Add `-DNDEBUG` to see the SIGSEGV instead of the abort.

**Files:**
- `xo-tokenizer/CMakeLists.txt:23` — the disabling comment
- `xo-tokenizer/include/xo/tokenizer/input_state.hpp:250` — `advance_until`,
  where the invariant breaks
- `xo-tokenizer/utest/tokenizer.test.cpp:215` — the failing case; `:385` is
  where the masked bug segfaults
- `xo-tokenizer/utest/span.test.cpp` — currently dead weight

**Done when:**
- the `input_state` invariant holds for every row of `s_testcase_v`
- `add_subdirectory(utest)` is uncommented
- `ctest --test-dir .build -R utest.tokenizer` passes, with asserts ENABLED
- a build with `-DNDEBUG` also passes (no latent out-of-range access)

## Notes

Do not "fix" this by deleting the assertion or the failing row. The assertion
is the only thing standing between this bug and a segfault, and the tokenizer
is load-bearing for xo-reader / xo-interpreter.

Related: `xo-tokenizer2` is a separate, active subsystem (`.forgejo` builds it
at line 489); this ticket is about the original `xo-tokenizer` only.
