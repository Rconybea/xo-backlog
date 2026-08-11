# 01 — utest.jit: MachPipeline failures after restore to top-level build

Status: fixed 2026-08-11
Type: bug

`utest.jit` was red, with two failures of different character. Both are fixed.
**Three separate defects were behind them, in three different subsystems**, and
only one was in xo-jit.

| Test | Symptom | Cause | Fixed in |
|---|---|---|---|
| `machpipeline.fptr` | 6/6 — exception escapes | nested lambda re-parented to the global env | `xo-jit/src/jit/MachPipeline.cpp` |
| `machpipeline.struct` | ~8% — SIGSEGV, plus **silently wrong results on every run** | `FunctionTdxInfo::operator==` inverted | `xo-reflect/include/xo/reflect/TypeDescr.hpp` |
| (both) | latent | tests called jit'd functions without the closure env argument | `xo-jit/utest/MachPipeline.test.cpp` |

Verification: **250 consecutive runs of `utest.jit`, 0 failures**, against 20/250
on the immediately preceding build. `xo-build --sweep` unchanged otherwise.

## 1. `machpipeline.fptr` — a nested lambda given the wrong parent

`MachPipeline::codegen_lambda_defn` did this, under a comment that states its own
precondition and does not check it:

```cpp
/* correct PROVIDED this is a toplevel lambda */
lambda->attach_envs(this->global_env_);
```

It is reached for **nested** lambdas too — `codegen_apply` → `codegen` →
`codegen_lambda_closure` → here — and a nested lambda already has a parent: the
`Lambda` constructor runs `body_->attach_envs(local_env_)`, which reaches a
lambda in function position. `LocalSymtab::assign_parent` correctly refuses the
second, conflicting parent, and the `std::runtime_error` escaped the test:

```
LocalSymtab::assign_parent(P2): already have established parent P1
  :P1 <LocalSymtab :argv "[<Variable :name x2 :type double>]">   <- root_2x's env
  :P2 <GlobalEnv :size 2>
```

`root_2x_ast()` is the trigger: `def root_2x(x2) { twice(sqrt, x2) }` puts the
lambda `twice` in function position.

**Fix:** ask instead of assume. `LocalSymtab::has_parent()` and
`Lambda::has_lexical_parent()` are new; the call site is now

```cpp
if (!lambda->has_lexical_parent())
    lambda->attach_envs(this->global_env_);
```

Skipping is not merely tolerable, it is correct: overwriting would discard the
enclosing frame, which is what `codegen_toplevel`'s `#ifdef OBSOLETE` block
already warns about ("don't do this anymore, obscures lexical context").

## 2. The tests called jit'd functions with the wrong signature

Every jit'd lambda takes its closure environment as an implicit **first**
argument:

```llvm
define double @root4(ptr %.env, double %x)
define %"ratio<int>" @make_ratio(ptr %.env, i32 %n, i32 %d)
```

`machpipeline.wrap` already called with it (`double(*)(void*, double)`); the
other two did not. Both call sites now pass `nullptr` for the environment.

**`machpipeline.fptr` was passing by ABI accident.** On x86-64 SysV a pointer
argument travels in an integer register and a `double` in an SSE register, so
the missing leading pointer never displaced `input`. It would displace an
integer argument — and did, in `machpipeline.struct`, where `%n` read the
register holding 2 and `%d` read an uninitialized one.

**`machpipeline.struct` never asserted its result.** It computed
`make_ratio(2,3)`, logged `:value.num 1 :value.den 0`, and reported success.
The test now requires 2 and 3. Worth stating plainly: a rendering-only
"assertion" is not an assertion, and this one hid a wrong answer through every
green run the ticket ever recorded.

## 3. The SIGSEGV: `FunctionTdxInfo::operator==` was inverted

The remaining ~8% crash was **not** in xo-jit. In
`xo-reflect/include/xo/reflect/TypeDescr.hpp`:

```cpp
inline bool operator==(const FunctionTdxInfo & other) const noexcept {
    if (retval_td_ != other.retval_td_)
        return true;              // <-- must be false
```

Two function types compared **equal** whenever their return types **differed**.

`TypeDescrBase::require_by_fn_info` uniques manually-constructed function types
through `static std::unordered_map<FunctionTdxInfo, TypeDescrBase *>
s_function_type_map`. `operator==` is consulted only on a **hash-bucket
collision**, and `std::hash<FunctionTdxInfo>` hashes over `TypeDescr`
**addresses** ("we can hash on addresses, since TypeDescr objects are
immutable"). Addresses move with ASLR. So:

- most runs — the two keys land in different buckets, `operator==` is never
  called, and everything is correct
- ~8% of runs — they collide, `operator==` says "equal" precisely because the
  return types differ, and `require_by_fn_info` returns **the wrong function
  type**

What that produced, caught by diffing the generated IR between a crashing and a
passing run of the same binary:

```llvm
correct:   define %"ratio<int>" @make_ratio(ptr %.env, i32 %n, i32 %d)
crashing:  define double        @make_ratio(ptr %.env, { ptr, ptr } %n, double %d)
```

The crashing signature is **`@twice`'s**, from `machpipeline.fptr`, which runs
first and shares the process-wide map. The generated code then stores a
`{ptr,ptr}` and a `double` — 16 bytes — into a frame consisting of one `push`:

```
push   %rax                 ; 8-byte frame
mov    %rsi,(%rsp)
mov    %rdx,0x8(%rsp)       ; the return address
movq   %xmm0,0x4(%rsp)      ; straddles it
...
pop    %rcx
ret                         ; -> smashed address
```

Hence `rip=0` with an empty backtrace.

**Fix:** `return false;`.

**Blast radius is much wider than this test.** `require_by_fn_info` is how every
manually-constructed function type is interned — every `Lambda` in xo-expression
goes through `assemble_lambda_td`. Any two function types differing only in
return type could be conflated, at a rate set by heap layout. Nothing else in
the tree failed after the fix (`xo-build --sweep`), but absence of a red test is
weak evidence here, by exactly the argument this ticket is about.

## How it was found, and what generalises

Four beliefs held along the way were wrong, and each was cheap to check:

1. **"~1 in 6, then ~2%."** Small samples. Measured over 250 runs it was
   **8%**, stable. Two of the intermediate readings (1/20, 2/82) were used to
   argue a fix had worked and later that a change had made things worse; both
   conclusions were noise. **Do not compare rates measured at n≈20.**
2. **"The absolute symbol resolves wrong."** `intern_symbol` interns the C++
   function with a default-constructed `JITSymbolFlags`, so it shows as
   `[Data][Hidden]` beside the jit's `[Callable]` entries. Plausible, and the
   theory was that a call to it could be resolved as a direct branch out of
   rel32 range. **Measured neutral: 20/250 with and without.** The flag change
   is kept — describing a function as data is wrong on its own terms — but it
   fixes nothing.
3. **"It's a memory bug; run ASan."** The ticket's own advice, and it would not
   have found this. The corruption is real but it is a *downstream* symptom of
   a wrong LLVM type.
4. **"gdb will show it."** 120 runs under gdb, zero reproductions — because gdb
   disables ASLR by default, and ASLR *is* the variable. `set
   disable-randomization off` reproduced it in 11.

What actually found it: **diffing the generated IR between a crashing and a
passing run of the same binary.** The moment the two were side by side the
signature was visibly another function's, and the search narrowed from "some
memory bug in a JIT" to "who hands out function types".

The debug build needed for that used the new passthrough:

```bash
xo-build --clobber --with-utests --with-examples --configure --build xo-jit \
  -- -DCMAKE_BUILD_TYPE=RelWithDebInfo
```

The default build has no `-g`, which is why every earlier backtrace was empty.

## Files

- `xo-reflect/include/xo/reflect/TypeDescr.hpp` — `FunctionTdxInfo::operator==`,
  and `std::hash<FunctionTdxInfo>` just below it
- `xo-reflect/src/reflect/TypeDescr.cpp:196` — `require_by_fn_info`
- `xo-jit/src/jit/MachPipeline.cpp` — the guarded `attach_envs`
- `xo-jit/include/xo/jit/Jit.hpp` — `intern_symbol` flags
- `xo-expression/include/xo/expression/{LocalSymtab,Lambda}.hpp` — the new
  `has_parent()` / `has_lexical_parent()` predicates
- `xo-jit/utest/MachPipeline.test.cpp` — both call sites, and the new assertions

## Done when

- [x] `utest.jit` passes 20 consecutive runs — passed **250**
- [x] `ctest --test-dir .build` green with no `-E` exclusion
- [x] the struct test asserts its result rather than logging it

## Follow-ups worth their own tickets

- **`std::hash<FunctionTdxInfo>` hashes addresses.** Correct for lookup, but it
  makes bucket assignment ASLR-dependent, which is what turned a plain logic bug
  into a 1-in-12 heisenbug. A content-based hash (canonical name, or TypeId
  rather than pointer) would make map behaviour reproducible across runs.
  Related: `.xo-backlog/process-counters/issues/01`, which is about the same
  class of run-to-run variation.
- **`FunctionTdxInfo::operator==` had no unit test.** xo-reflect has no test
  that two function types differing only in return type compare unequal — the
  one assertion that would have caught this directly.
- **No sanitizer knob for UBSan.** `XO_ADDRESS_SANITIZE` exists
  (`xo-cmake/cmake/xo_macros/xo_cxx.cmake:771-792`); a matching
  `XO_UNDEFINED_SANITIZE` is a small change to the same block.
