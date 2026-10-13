# std-function-reflection — reflect std::function as a function type

Status: open
Type: spec
Milestone: reflection-driven-json
Ticket: `.xo-backlog/xo-reflect/issues/09-std-function-reflection.md`

Teach xo-reflect `std::function<R(Args...)>`: its signature at the type
level, whether a value is set at the value level.  printjson then prints a
`std::function` (and a function pointer) generically, and the last bespoke
websock struct printer -- `JsonPrinter_DynamicEndpoint`, kept only to
summarise its three `std::function` members -- retires
(`.xo-backlog/xo-websock/issues/16`).

## Purpose and scope (RC, 2026-10-12/13)

- **A callable is a foundational kind of value.**  One metatype means
  "callable": `mt_function`.  A `std::function` is one; it is NOT a new
  metatype.  Where consumers need to tell kinds apart, that is an
  `mt_function`-only detail (RC: "pick up the existing mt_function
  infrastructure, maybe generalize if necessary").
- **Consumer: introspection, for now.**  printjson and the introspect page.
  The signature is captured so others can use it later; only the printing
  path is built and tested.
- **A value prints `"set"` / `"empty"`.**  Not the target's `type_info`
  (compiler-spelled closure names), not a function pointer's symbol.  The
  signature is visible through the declared type (`_canonical_type_`).
- **`std::function` only.**  `std::move_only_function`, `std::function_ref`
  later, the same way.

## What exists (2026-10-13)

- `Metatype::mt_function` (`xo-reflect/include/xo/reflect/Metatype.hpp:9`);
  `FunctionTdx` (`include/xo/reflect/function/FunctionTdx.hpp`): return
  type, argument types, `noexcept`, through `fn_retval()`, `n_fn_arg()`,
  `fn_arg(i)`, `fn_is_noexcept()`, `fn_info()`.
- Function POINTERS only: `EstablishTdx<Retval (*)(Args...)>` and a
  `noexcept` variant (`include/xo/reflect/Reflect.hpp:249-258`, defined
  `:686`), via `detail::AssembleArgv<Args...>`.
- **Interning by ingredients.**  `TypeDescrBase::require` files every
  function `TypeDescr` in `s_function_type_map`, keyed by `FunctionTdxInfo`
  (return type, argument types, `noexcept`)
  (`src/reflect/TypeDescr.cpp:194-195`); `require_by_fn_info` (`:202`) looks
  a signature up there, else builds one with no native type, named by
  `FunctionTdxInfo::make_canonical_name()` (`:31`, spelled `R (*)(A,..)`).
  Purpose (RC): keep `TypeDescrTable` from growing duplicates -- xo-reflect's
  tables have no recovery story.
- No `EstablishTdx` for `std::function`
  (`grep -rn 'EstablishTdx<std::function' xo-reflect/include`: nothing), so a
  `std::function` member is an opaque atomic, and printjson writes
  `<error-json-printer-not-found...>`.
- printjson cannot print ANY `mt_function` value: `print_node` writes
  `<error-json-printer-not-implemented ...>`
  (`xo-printjson/src/printjson/JsonPrintState.cpp:619-625`).
- `is_function()` consumers outside xo-reflect -- all treat it as "a raw
  function that code can call":

  ```bash
  grep -rn -E 'is_function\(\)' --include=*.cpp --include=*.hpp xo-expression xo-jit
  ```

  `xo-expression/src/expression/Apply.cpp:25`,
  `xo-expression/include/xo/expression/PrimitiveExpr.hpp:117`,
  `xo-jit/src/jit/type2llvm.cpp:32`.  (`xo-jit/src/jit/MachPipeline.new.cpp`,
  `.orig.cpp` also call it but are not compiled -- only `MachPipeline.cpp` is
  in `xo-jit/src/jit/CMakeLists.txt` -- and are left alone;
  `xo-expression/example/ex1/ex1.cpp:38` only prints it.)

## Design

### 1. Type level: a representation, part of the interning key

- **`FunctionRepr { pointer, std_function }`** -- a new field of
  `FunctionTdxInfo`.  Both are `mt_function`.
  - It joins `FunctionTdxInfo::operator==`, the `std::hash` specialisation
    (`TypeDescr.hpp:682`) and `make_canonical_name()` -- so a
    `std::function<R(Args...)>` and an `R (*)(Args...)`, same signature,
    different C++ types, intern SEPARATELY.  Without it the second
    registered would overwrite the first's `s_function_type_map` entry, and
    `require_by_fn_info` would return the wrong type.
  - `pointer` is today's `R (*)(Args...)`; existing function types default
    to it.  `make_canonical_name()` spells `std_function` as
    `std::function<R (A,..)>` (used only on the ingredient path; natively
    reflected types keep their `type_name<T>` name).
- **`EstablishTdx<std::function<R(Args...)>>`**: a partial specialisation,
  deducing `R`, `Args...` with `detail::AssembleArgv`, building a
  `FunctionTdx` of repr `std_function`, `noexcept` false (a `std::function`
  signature cannot be `noexcept`).
- **The signature needs no new API**: a `std::function` type's own
  `TypeDescr` answers `fn_retval()`, `n_fn_arg()`, `fn_arg(i)`,
  `fn_is_noexcept()`.
- **One new accessor, `fn_repr()`**: a virtual on `TypeDescrExtra` (default
  `pointer`), overridden by `FunctionTdx`, passed through `TypeDescr` --
  exactly as `fn_retval()`, `n_fn_arg()`, `fn_arg(i)`, `fn_is_noexcept()`
  are (`TypeDescrExtra.hpp:131-134`, `TypeDescr.hpp:444-451`).  The
  `mt_function`-only detail consumers check.
- **No second route to a `std_function` entry**: `require_by_fn_info`
  accepts only `repr == pointer` (asserts otherwise).  A `std::function`
  always has a native C++ type, so it enters only through
  `Reflect::require<T>`, once.  (The analogous pointer hazard -- built from
  ingredients first, then reflected natively, gets a second entry, since the
  native path looks up by `typeid` -- exists today and is unchanged here.)

### 2. Values: set or empty; printing

- **`bool FunctionTdx::fn_is_set(void * object) const`**:
  - `repr == pointer`: generic -- a function pointer's storage is a function
    pointer whatever its signature: read it
    (`*reinterpret_cast<void (* const *)()>(object)`) and compare with
    `nullptr`.  No helper; works for ingredient-built pointer types too.
  - `repr == std_function`: a `bool (*)(void *)` supplied by
    `EstablishTdx<std::function<R(Args...)>>` -- `static_cast<bool>(f)`; the
    one kind whose emptiness needs the concrete type.
- **printjson**: the `mt_function` case of `print_node` prints `"set"` or
  `"empty"` from `fn_is_set`, for both reprs.  So function pointers stop
  printing `<error-json-printer-not-implemented...>` too -- no extra cost, the
  same value shape.

### 3. Consumers that call code: guard on `fn_repr()`

Each site keeps its meaning explicit:
`td->is_function() && td->fn_repr() == FunctionRepr::pointer`; a
`std::function` takes the site's existing "not a function" path, with a
message naming why ("a std::function is not supported here; a function
pointer is").  No `std::function` reaches these today -- defensive.  No
convenience such as `is_function_pointer()` on every `TypeDescr`.

- `Apply.cpp:25`, `PrimitiveExpr.hpp:117` (xo-expression): **a deferral.**
  An interpreter could call a `std::function` primitive through its
  `operator()`; the guard marks a capability not yet built (RC).
- `type2llvm.cpp:32` (xo-jit): **likely permanent.**  A type-erased object
  is opaque to compiled code (RC).  If the `if / else if` chain's final
  branch is not already an error, the `std_function` case throws there
  rather than produce a layout.

Unaffected: xo-pyreflect (the metatype is unchanged); other printjson code.

### 4. DynamicEndpoint, cleanup

- `DynamicEndpoint::reflect_self` adds `http_handler`, `subscribe_fn`,
  `unsubscribe_fn`; `JsonPrinter_DynamicEndpoint` is deleted.  The endpoint
  prints from reflection: its members lose the trailing `_`, as every other
  websock type's did (`xo-websock/issues/16`); the page's three reads
  (`kind_`, `uri_pattern_`, `receiver_`) and the tests rename with them.
  The three functions print `"set"` / `"empty"` through section 2.
- `provide_websock_json_printers` then registers nothing: deleted (RC), with
  `src/websock/websock_json.cpp` (nothing else left in it), the installed
  `include/xo/websock/websock_json.hpp`, its one call
  (`src/websock/WebsockAppcx.cpp:25`), and their CMake entries.
- `.xo-backlog/xo-websock/issues/16` closes with no recorded exceptions.

## Tests

- **xo-reflect** (`utest/FunctionTdx.test.cpp`):
  - `std::function<int(double)>`: `mt_function`, `fn_repr() ==
    std_function`, `fn_retval()` `int`, `n_fn_arg()` 1, `fn_arg(0)` `double`;
  - a DIFFERENT `TypeDescr` from `int (*)(double)`, and registering it does
    not overwrite the pointer's entry: `require_by_fn_info` (pointer repr)
    still returns the pointer type afterwards -- the section 1 hazard;
  - `fn_is_set`: set / empty `std::function`; non-null / null function
    pointer, native and ingredient-built.
- **xo-printjson**: a struct holding a set `std::function`, an empty one and
  a function pointer prints `"set"` / `"empty"` for each, with each member's
  `_canonical_type_` spelling its declared type.
- **xo-expression**: `PrimitiveExpr` given a `std::function` type throws,
  naming function pointers -- if its tests can construct one cheaply (decide
  when planning).
- **xo-jit**: the guard is not tested (no path reaches `type2llvm` with a
  `std::function` today); a comment there says so.
- **xo-websock**: both golden files (expected: exactly the endpoint's renames
  and `_name_` -> its short type, checked by script), the C++ and browser
  tests, gcc and clang.

## Order

xo-reflect, xo-printjson, xo-expression, xo-jit (guard), xo-websock -- the
dependency order; each its own commit.

**Done when:** a `std::function` member reflects with its signature and
prints through printjson's generic path; `JsonPrinter_DynamicEndpoint` and
`provide_websock_json_printers` are gone; `xo-reflect/issues/09` and
`xo-websock/issues/16` are done.
