# std-function-reflection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** reflect `std::function<R(Args...)>` as an `mt_function` (signature at the type level, set/empty at the value level), print function values generically, and retire the last bespoke websock printer.

**Architecture:** `std::function` joins the existing `mt_function` machinery (`FunctionTdx`), distinguished by a `FunctionRepr` that is part of the function-type interning key.  A per-value `fn_is_set` lets printjson print any function as `"set"` / `"empty"`.  Code-calling consumers (xo-expression, xo-jit) guard on `fn_repr()`.

**Tech Stack:** C++20 (gcc 14.3 and clang 18.1.8, `-Werror -Wall -Wextra`), Catch2 v2, cmake, nix; the introspect page's browser tests (node 22 + headless chrome).

**Spec:** `.xo-backlog/std-function-reflection/spec.md` (approved by RC 2026-10-13).  Read it first; this plan argues from it.

**How this plan is written:** behaviour, interfaces, invariants and named risks -- NOT function bodies (a plan carrying literal code blinds per-task review: reviewer and implementer would read the same source).  Each task's implementer writes the code; each reviewer checks it against this plan and the spec.

## Global Constraints

- **Never `git commit` / `git push` in the umbrella repo** (`~/proj/xo-umbrella2`).  Each task ends by stopping for RC to review and commit.  Backlog tickets (`~/proj/xo-backlog`, the `.xo-backlog` symlink) may be committed and pushed.
- **Build the umbrella tree:** `cmake --build .build -j16` from `~/proj/xo-umbrella2`.  `-Werror` is on: no new warnings.
- **Build under CI's clang:** `nix-shell ci-nxfs.nix -A xo-builder-shell --run 'cmake --build .build-clang -j16'` (`.build-clang` exists, configured with `CC=clang CXX=clang++`; see `.xo-backlog/ci/issues/03`).
- **Run a suite:** the test binary directly, e.g. `.build/xo-reflect/utest/utest.reflect`; `ctest --test-dir .build/<subsystem>` for one subsystem.  Never hide output with `>/dev/null` or `2>/dev/null` on the line whose result you report.
- **Include style:** same-repo headers with `"quotes"`, cross-subsystem headers with `<angle>` brackets.
- **`using` declarations:** at namespace scope in `.cpp` only; in headers, inside function bodies -- never leaking into includers.
- **Messages:** compose strings with `xo::tostr(...)`, not `std::string` `operator+`.
- **Comments:** match the surrounding density; cite the ticket (`.xo-backlog/xo-reflect/issues/09`) where a reader would need the reason.
- **Golden files** (`xo-websock/utest/golden/server-snapshot.{gcc,clang}.json`) are rewritten only with `XO_UPDATE_GOLDEN=1`, gcc under `.build`, clang under `.build-clang` inside the builder shell -- and only after a script confirms the diff is exactly the intended change (Task 4).
- **Browser tests** serve a COPY of `mount-origin/` made at build time: rebuild `websock_ex_introspect` before `xo-websock/example/introspect/test/run.sh .build/xo-websock/example/introspect/websock_ex_introspect`.

## Review Focus

The inputs most likely to bite that the spec implies but does not name, most likely first.  Each has its test in the owning task (marked **[RF n]**).

1. **Reference and `void` in a signature.**  DynamicEndpoint's signatures are `HttpResponse (const HttpRequest&)`, `CallbackId (const rp<WebsocketSink>&)`, `void (CallbackId)`.  Whether `detail::AssembleArgv` / `Reflect::require` describe a `const T&` argument and a `void` return the way they do for function pointers is unverified.  Expect: `fn_arg(i)` and `fn_retval()` are the same `TypeDescr`s a function pointer with that signature reports.  Task 1.
2. **Registration order.**  The interning hazard bites whichever of `R (*)(A)` and `std::function<R(A)>` registers SECOND.  Expect: both orders leave two distinct entries, and `require_by_fn_info` (pointer repr) returns the pointer type afterwards.  Task 1 (both orders, different signatures so they don't share state).
3. **A `std::function` not inside a struct.**  A vector element (and a top-level print) reaches `print_node`'s `mt_function` case directly, not through `JsonMembers`.  Expect: `"set"` / `"empty"` there too.  Task 2.
4. **Compiler spelling.**  A `std::function`'s canonical name is compiler-spelled (`type_name<T>`) and differs between gcc and clang.  Expect: tests compare against `xo::reflect::type_name<T>()`, never a literal string; both golden files update.  Tasks 2, 4.
5. **Emptying after construction.**  A `std::function` assigned `nullptr` after holding a callable.  Expect: `fn_is_set` false (the value is read at print time, not cached from reflection).  Task 1.

---

### Task 1: xo-reflect -- reference types; `FunctionRepr`, `std::function` reflection, `fn_is_set`

(Widened 2026-10-14, RC: spec section 1b, reference types -- found
implementing; they come first, since the `std::function` signatures need
them.)

**Files:**
- Create: `xo-reflect/include/xo/reflect/pointer/PointerRepr.hpp` (the enum only)
- Create: `xo-reflect/include/xo/reflect/pointer/ReferenceTdx.hpp` (a `PointerTdx`, beside `RawPointerTdx`)
- Modify: `xo-reflect/include/xo/reflect/pointer/PointerTdx.hpp` (`pointer_repr()` on each existing class)
- Modify: `xo-reflect/include/xo/reflect/EstablishTypeDescr.hpp` (`establish<T>` at :41-45: a reference branch)
- Modify: `xo-reflect/include/xo/reflect/Reflect.hpp` (`EstablishTdx<T &>`, `EstablishTdx<T &&>`, beside `EstablishTdx<T *>` at :558-571)
- Test: `xo-reflect/utest/ReferenceTdx.test.cpp` (new; add to `xo-reflect/utest/CMakeLists.txt`)
- Create: `xo-reflect/include/xo/reflect/function/FunctionRepr.hpp` (the enum only)
- Modify: `xo-reflect/include/xo/reflect/TypeDescr.hpp` (`FunctionTdxInfo` at :164-207; its `std::hash` at :682; `TypeDescr` pass-throughs beside :444-451)
- Modify: `xo-reflect/src/reflect/TypeDescr.cpp` (`FunctionTdxInfo::make_canonical_name` at :31; `require_by_fn_info` at :202)
- Modify: `xo-reflect/include/xo/reflect/TypeDescrExtra.hpp` (function virtuals beside :130-134)
- Modify: `xo-reflect/include/xo/reflect/function/FunctionTdx.hpp`, `xo-reflect/src/reflect/function/FunctionTdx.cpp`
- Modify: `xo-reflect/include/xo/reflect/Reflect.hpp` (new specialisation beside the function-pointer ones at :245-262 and their definitions at :686-705)
- Test: `xo-reflect/utest/FunctionTdx.test.cpp`

**Interfaces:**
- Produces:
  - `enum class PointerRepr { raw, smart, lvalue_reference, rvalue_reference };` in `xo::reflect`; `virtual PointerRepr TypeDescrExtra::pointer_repr() const` (default `raw`), `TypeDescrBase::pointer_repr()` pass-through; `RawPointerTdx` -> `raw`, `RefPointerTdx` -> `smart`, `ReferenceTdx` -> its kind.
  - `Reflect::require<T &>()` / `<T &&>()`: `mt_pointer`, a `ReferenceTdx`, no native typeinfo, keyed by `type_name<T &>()`.
  - `enum class FunctionRepr { pointer, std_function };` in namespace `xo::reflect`.
  - `FunctionTdxInfo` gains a `FunctionRepr repr_` member, defaulting to `pointer`; its existing 3-argument constructor `(TypeDescr retval_td, const std::vector<TypeDescr> & arg_td_v, bool is_noexcept)` keeps working unchanged (callers: `xo-expression/src/expression/Lambda.cpp:40`, `xo-expression2/src/expression2/DLambdaExpr.cpp:113`); a 4-argument form adds `repr`.  `operator==` and the hash include `repr_`.
  - `virtual FunctionRepr TypeDescrExtra::fn_repr() const` (default `pointer`); `virtual bool TypeDescrExtra::fn_is_set(void * object) const` (default `false`); `FunctionTdx` overrides both.
  - `TypeDescrBase::fn_repr() const`, `TypeDescrBase::fn_is_set(void * object) const` -- pass-throughs, as `fn_retval()` is.
  - `EstablishTdx<std::function<R(Args...)>>` -- a partial specialisation.
- Consumes: nothing new.

**Behaviour and invariants:**
- **References first** (spec 1b): `ReferenceTdx` -- pointee the cv-stripped referent (`fixed_child_td(0)`), `child_edge_ownership()` `borrowed`, value operations (`n_child`, `child_tp`) throw ("references are described, not traversed").  `establish<T &>` / `establish<T &&>` pass no `&typeid` and no invoker, so the entry never collides with the referent's (`typeid(T&) == typeid(T)`).
- `make_canonical_name()`: `pointer` unchanged (`R (*)(A,..)`); `std_function` spelled `std::function<R (A,..)>`.
- `require_by_fn_info(info)` asserts `info.repr_ == FunctionRepr::pointer`: a `std::function` type enters only through `Reflect::require<T>` (spec section 1).
- `EstablishTdx<std::function<R(Args...)>>` builds a `FunctionTdx` with repr `std_function`, `noexcept` false, return/argument types as the function-pointer specialisation computes them (reuse `detail::AssembleArgv<Args...>`), and a `bool (*)(void *)` that answers `static_cast<bool>(*static_cast<std::function<R(Args...)> const *>(object))`.
- `FunctionTdx::fn_is_set(object)`: `pointer` -- read `*reinterpret_cast<void (* const *)()>(object)`, compare with `nullptr`, no helper (works for ingredient-built types); `std_function` -- call the helper.  A `FunctionTdx` of repr `std_function` with no helper is a construction bug: assert.
- `std::function`'s metatype is `mt_function`; `n_child` stays 0.

- [ ] **Step 0: references, test first** in `ReferenceTdx.test.cpp` (tag `[reflect][reference]`): `const std::string &` -- `mt_pointer`, `pointer_repr() == lvalue_reference`, `fixed_child_td(0) == require<std::string>()`, canonical name `== type_name<const std::string &>()`, `native_typeinfo() == nullptr`; `std::string &&` -- `rvalue_reference`; required twice -> same `TypeDescr`; differs from `require<std::string>()` and `require<std::string *>()`, and `require<std::string>()` is unchanged afterwards (still its own `typeid`); `pointer_repr()` of `int *` -- `raw`, of `std::unique_ptr<int>` -- `smart`; `n_child` on a reference type throws.  Run (`.build/xo-reflect/utest/utest.reflect "[reference]"`), see it fail, implement the reference half, see it pass.
- [ ] **Step 1: Write the failing tests** in `FunctionTdx.test.cpp` (beside the existing function-pointer case at :25-35).  Each is its own `TEST_CASE`:
  - *std-function-reflects-as-mt-function:* `Reflect::require<std::function<int (double)>>()` -- `metatype() == mt_function`, `fn_repr() == FunctionRepr::std_function`, `fn_retval() == Reflect::require<int>()`, `n_fn_arg() == 1`, `fn_arg(0) == Reflect::require<double>()`, `fn_is_noexcept()` false.
  - *std-function-and-pointer-intern-separately (pointer first)* **[RF 2]**: require `long (*)(short)`, then `std::function<long (short)>`; the two `TypeDescr`s differ; `require_by_fn_info(FunctionTdxInfo(require<long>, {require<short>}, false))` returns the POINTER's.
  - *... (std::function first)* **[RF 2]**: the same with another signature (`char (float)`), registering `std::function` first.
  - *signature-with-reference-and-void* **[RF 1]**: `std::function<void (std::string const &)>` and `void (*)(std::string const &)` report equal `fn_retval()` and equal `fn_arg(0)`.
  - *fn-is-set:* a `std::function<int (double)>` default-constructed -- false; holding a lambda -- true; then assigned `nullptr` -- false **[RF 5]**; a function pointer `nullptr` -- false, pointing at a function -- true; an ingredient-built pointer type (`require_by_fn_info`) on a function pointer -- true.
- [ ] **Step 2: Run them and see them fail:** `cmake --build .build --target utest.reflect -j16` then `.build/xo-reflect/utest/utest.reflect "[function]"` (tag the new cases `[reflect][function]`).  Expected: compile errors first (`FunctionRepr`, `fn_repr`, `fn_is_set` undeclared) -- that is the failure; there is no `std::function` specialisation yet.
- [ ] **Step 3: Implement** per *Behaviour and invariants*.
- [ ] **Step 4: Run them and see them pass**, then the whole suite: `.build/xo-reflect/utest/utest.reflect` -- all pass, no new warnings.
- [ ] **Step 5: Build everything** (`cmake --build .build -j16`): xo-expression's and xo-expression2's ingredient-path callers still compile unchanged.  Then the clang tree.
- [ ] **Step 6: Stop for RC to review and commit.**

### Task 2: xo-printjson -- print a function value as set / empty

**Files:**
- Modify: `xo-printjson/src/printjson/JsonPrintState.cpp` (the `mt_function` case of `print_node`, :619-625)
- Create: `xo-printjson/utest/PrintJsonFunction.test.cpp`
- Modify: `xo-printjson/utest/CMakeLists.txt` (add the new test file, as `PrintJsonMap.test.cpp` was added)

**Interfaces:**
- Consumes: `TypeDescrBase::fn_is_set(void *) const` (Task 1).
- Produces: a function value prints as the json string `"set"` or `"empty"`.

**Behaviour:** the `mt_function` case writes `"set"` / `"empty"` from `tp.td()->fn_is_set(tp.address())`, for both reprs, replacing `<error-json-printer-not-implemented ...>`.  Function values carry no `_id_` (they are values, like atomics).

- [ ] **Step 1: Write the failing tests** (tag `[printjson][function]`):
  - *function-members-print-set-or-empty:* a reflected test struct with a set `std::function<int (double)>`, an empty one, a non-null `int (*)(double)` and a null one; print it; each member's `_value_` is `"set"` / `"empty"` as expected; each member's `_canonical_type_` equals `xo::reflect::type_name<T>()` for its declared type **[RF 4]**; the output parses as json (no `<error-` text).
  - *a-vector-of-std-functions:* a `std::vector<std::function<void ()>>` holding one set and one empty, printed on its own **[RF 3]** -- `["set", "empty"]`.
- [ ] **Step 2: Run and see them fail:** `cmake --build .build --target utest.printjson -j16`, `.build/xo-printjson/utest/utest.printjson "[function]"` -- expected: the `<error-json-printer-not-implemented` text where `"set"`/`"empty"` should be.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run them and the whole suite** (`.build/xo-printjson/utest/utest.printjson`) -- pass.  Then the clang tree.
- [ ] **Step 5: Stop for RC to review and commit.**

### Task 3: xo-expression, xo-jit -- guard code-calling consumers on `fn_repr()`

**Files:**
- Modify: `xo-expression/src/expression/Apply.cpp:25`
- Modify: `xo-expression/include/xo/expression/PrimitiveExpr.hpp:117`
- Modify: `xo-jit/src/jit/type2llvm.cpp:32`

**Interfaces:** Consumes `TypeDescrBase::fn_repr()` and `FunctionRepr` (Task 1).

**Behaviour:**
- Each site's function test becomes `is_function() && fn_repr() == FunctionRepr::pointer`.
- `Apply` and `PrimitiveExpr`: a `std::function` takes the existing throw, its message saying a `std::function` is not supported here, a function pointer is.  A comment marks it a **deferral** (an interpreter could call a `std::function` primitive -- spec section 3).
- `type2llvm`: a `std::function` falls to the chain's final branch, which already prints `td_to_llvm_type: no llvm type available for T` and returns `nullptr` (`type2llvm.cpp:56-60`) -- the existing unsupported-type signal, so no new throw.  A comment marks it **likely permanent** (opaque to compiled code).
- Do not touch `xo-jit/src/jit/MachPipeline.new.cpp` / `.orig.cpp` (not compiled) or `xo-expression/example/ex1/ex1.cpp` (prints only).

**No new tests, deliberately:** no `std::function` reaches these sites today, and a `std::function` `PrimitiveExpr` cannot even be instantiated (its `pretty()` casts the value `(void*)(this->value())`, `PrimitiveExpr.hpp:101`).  The spec makes the xo-expression test conditional on being cheap; it is not.

- [ ] **Step 1: Implement the three guards and their comments.**
- [ ] **Step 2: Build everything and run the existing suites:** `cmake --build .build -j16`; `ctest --test-dir .build/xo-expression`, `ctest --test-dir .build/xo-jit` -- unchanged results.  Then the clang tree.
- [ ] **Step 3: Stop for RC to review and commit.**

### Task 4: xo-websock -- DynamicEndpoint prints from reflection; delete the last printer

**Files:**
- Modify: `xo-websock/src/websock/DynamicEndpoint.cpp` (`DynamicEndpoint::reflect_self` and its comment, ~:155-171)
- Delete: `xo-websock/src/websock/websock_json.cpp`, `xo-websock/include/xo/websock/websock_json.hpp`
- Modify: `xo-websock/src/websock/CMakeLists.txt:14` (drop `websock_json.cpp`); `xo-websock/src/websock/WebsockAppcx.cpp` (drop the include at :7 and the call at :25)
- Modify (comments naming the deleted header): `xo-websock/include/xo/websock/cx/WebsockAppcx.hpp:19`, `xo-websock/utest/Webserver.test.cpp:47`
- Modify: `xo-websock/example/introspect/mount-origin/introspect.js` (reads of `kind_`, `uri_pattern_`, `receiver_`; its header comment), the C++ tests and browser tests that name the endpoint's members, both golden files

**Interfaces:** Consumes Task 1 (a `std::function` member is reflectable) and Task 2 (it prints).

**Behaviour:**
- `REFLECT_MEMBER` for `http_handler`, `subscribe_fn`, `unsubscribe_fn`; the comment no longer says they are unreflectable.
- The endpoint prints with the generic printer: `_name_` its short type (`DynamicEndpoint`, unchanged); members renamed without the trailing `_`: `kind`, `uri_pattern`, `var_v`, `receiver`, `http_handler`, `subscribe_fn`, `unsubscribe_fn`, in reflection order.  Values unchanged (the functions still `"set"` / `"empty"`).
- After the deletions nothing names `provide_websock_json_printers` or `websock_json.hpp`: `grep -rn 'provide_websock_json_printers\|websock_json' xo-* --include=*.cpp --include=*.hpp --include=*.txt` (excluding `.build`) finds nothing.

- [ ] **Step 1: Implement the reflection and the deletions; build** (`cmake --build .build -j16`).  The full tree must build -- xo-reactor2websock and xo-pyreactor2websock depend on xo-websock.
- [ ] **Step 2: Run `utest.websock` and `utest.websock.live`; move each failing read onto the new names.**  Expected failures: the expected member list in `Webserver.test.cpp` (~:459) and its `value_of(m, "kind_")`-style reads; `WebserverLive.test.cpp`'s `uri_pattern_` read (~:836); the endpoint helpers' map/member reads.  Keep each check's intent.
- [ ] **Step 3: Audit the golden diff by script before updating it.**  Applying exactly "each `DynamicEndpoint`'s `_members_` names drop their trailing `_`" to the current `server-snapshot.gcc.json` must equal the new `server-snapshot.actual.json` (ids unchanged).  Any other difference is a bug, not an update.
- [ ] **Step 4: Update both golden files** (gcc in `.build`, clang in `.build-clang` in the builder shell) **[RF 4]**; confirm they are equal ignoring `_canonical_type_` / `_short_type_` and template `_name_`s.
- [ ] **Step 5: The page and browser tests.**  Rename the page's three reads and its header comment; rebuild `websock_ex_introspect`; run all browser tests; rename the endpoint-member strings they check (at least `endpoint_expand.mjs`, `receiver.mjs`, `rows.mjs`, `edge_kinds.mjs`, `anchor.mjs`, `visibility.mjs` -- grep `\b(kind|uri_pattern|var_v|receiver|http_handler|subscribe_fn|unsubscribe_fn)_\b`).  All 21 pass.
- [ ] **Step 6: Downstream:** `ctest --test-dir .build/xo-reactor2websock`, `.build/xo-pyreactor2websock`; clang: `utest.websock`, `utest.websock.live`.
- [ ] **Step 7: Stop for RC to review and commit.**

### Task 5: close out -- sweep, tickets

**Files:** `~/proj/xo-backlog`: `xo-reflect/issues/09-std-function-reflection.md`, `xo-websock/issues/16-retire-websock-printers.md`, `std-function-reflection/spec.md` (Status), `CONVENTIONS.md` only if the sweep's totals moved.

- [ ] **Step 1: The full sweep:** `xo-build --sweep` from `~/proj/xo-umbrella2` -- expect the closing line `xo-build: --sweep ok (build and utest)`.  If a total moved from CONVENTIONS' baseline, find out why before updating it.
- [ ] **Step 2: Mark xo-reflect/09 and xo-websock/16 done**, each with a Resolution naming the umbrella commits and what was verified; mark the spec `Status: done`.  `xo-sdlc --milestones` should then show reflection-driven-json with nothing open.  Commit and push the backlog.
