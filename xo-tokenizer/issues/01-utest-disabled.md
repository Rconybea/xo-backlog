# 01 — xo-tokenizer/utest is disabled; one real bug is keeping it out

Status: resolved (2026-08-06)
Type: bug
Milestone: ppsink-migration

## Resolution

Not one bug -- **seven**, plus one wrong guess by the estimate below. `utest.tokenizer`
is re-enabled and green: 401 assertions / 8 test cases, 20 consecutive clean runs,
and clean under `-DNDEBUG` (so no latent out-of-bounds access hides behind the
assertions). Full umbrella is 33/34, only `utest.jit` outstanding.

Six library defects, four of them the same shape -- pointer arithmetic on a span
without a bounds check:

| Site | Defect |
|---|---|
| `tokenizer.hpp` 2-char punct | end-of-input guard `#ifdef OBSOLETE`'d out on the false premise that input always ends in whitespace; `ix` advanced to `hi+1` |
| `input_state.hpp` `skip_leading_whitespace` | `is_whitespace(*ix) && (ix != hi)` -- deref *before* the bounds check |
| `input_state.hpp` `capture_current_line` | `if (*eol == '\n')` deref at `input.hi()` when the input has no newline |
| `tokenizer.hpp` `scan` | `capture_current_line()`'s `input_error` return discarded at its only call site, so a null `current_line_` reached a deref |
| `tokenizer.hpp` `scan` | `current_line_` never advanced past a newline, so a token on line 2 tripped `advance_until`'s assertion |
| `tokenizer_error.hpp` | `tk_start()` returned `current_pos()` -- the token's *end*, not its start |

Two unimplemented behaviours, resolved by design decision (RC):

- **`scan_result::consumed` was never populated on a token result** -- measured 0
  for all 71 testcases. `assemble_token` returned `current_line().prefix(0)`
  (always empty) under a "caller holds the whole line" policy, so `consume_all_`
  in the test table had never been exercised. Now a token-bearing result reports
  `[tk_start - whitespace, tk_end)` -- the whitespace skipped ahead of the token
  plus the token itself. This reproduces the table's subtle `{"> ", .., true}` vs
  `{"0 ", .., false}` distinction exactly.
- **Unterminated string literals** were diagnosed generically by `scan`
  ("unterminated string literal"), pre-empting `assemble_token`, which can say
  *why* -- dangling escape vs missing terminator. `scan` now defers. The naked
  newline/CR case stays in `scan`: `assemble_token` never sees across a line.

Two test bugs:

- `tkz.scan(in_span, in_span.empty())` passed `eof=false` for every non-empty
  input, so `capture_current_line` correctly returned `incomplete` and no token
  could ever be produced. Every assertion in `tokenizer2` sits behind
  `if (tk.is_valid())`, so **the whole case had been passing vacuously**.
- (none other -- the `consume_all_` expectations turned out to be correct, and
  it was the library that was missing the feature)

### Correction to the analysis below

The estimate "It is one failing test case, not a broken suite" was right about
the *symptom* and wrong about the cause: fixing each defect exposed the next.
The framing that mattered came from RC -- an input span is a block of text that
may cross several lines, and `current_line_` is error-reporting context ("used
only to report errors", `input_state.hpp`), NOT a scanning boundary. An earlier
attempt here bounded `scan` on the line and had to be reverted; the correct
change is that `current_line_` *advances* to track the line holding the scan
position, which is what `tokenizer.hpp`'s own comment had always claimed
happened but nothing implemented.

`input_state` gained one method, `add_whitespace()`, so a whitespace run
straddling a newline still counts as a single gap between tokens.

---

## Original analysis

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
