# 09 -- reflect std::function: its signature, and what a value can show

Status: open (design first)
Type: feature
Milestone: reflection-driven-json
Raised: 2026-10-12 (RC), from `.xo-backlog/xo-websock/issues/16` (DynamicEndpoint)

A callable is a foundational kind of value (RC): teach xo-reflect
`std::function<R(Args...)>`, so a struct holding one prints from
reflection instead of a bespoke printer summarising it.

## What exists (2026-10-12)

- `Metatype::mt_function` (`include/xo/reflect/Metatype.hpp:9`),
  `FunctionTdx` (`include/xo/reflect/function/FunctionTdx.hpp`).
- Function POINTERS are reflected with their signature:
  `EstablishTdx<Retval (*)(Args...)>` and a `noexcept` variant
  (`Reflect.hpp:249-258`, defined `:686`) build a `FunctionTdx` from the
  return type's `TypeDescr`, the argument types' (`detail::AssembleArgv`),
  and `noexcept`.  `FunctionTdxInfo` makes a signature hashable
  (`TypeDescr.hpp:164-207`); `require_by_fn_info` / `s_function_type_map`
  intern one `TypeDescr` per signature.
- `std::function` has no `EstablishTdx`:

  ```bash
  cd "$(xo-sdlc --umbrella2)" && grep -rn 'EstablishTdx<std::function' xo-reflect/include   # nothing
  ```

  so a `std::function` member falls to an opaque atomic, and printjson,
  finding no printer, writes `<error-json-printer-not-found...>` -- not
  even json (`xo-printjson/src/printjson/JsonPrintState.cpp`, `print_node`'s
  fallback).  Today `JsonPrinter_DynamicEndpoint` hand-summarises its three
  (`http_handler_`, `subscribe_fn_`, `unsubscribe_fn_`) as `"set"` /
  `"empty"` (`xo-websock/src/websock/websock_json.cpp`).

## What a std::function can show

- **Its type: the whole signature.**  A partial specialisation
  `EstablishTdx<std::function<R(Args...)>>` deduces `R`, `Args...` as the
  function-pointer case does, and can reuse the interned signature
  (`FunctionTdxInfo`).  Each argument and the return type is a full
  `TypeDescr`: navigable, and linkable to source by the introspect page.
- **A value: little -- `std::function` erases its target's type.**
  - set or empty: `static_cast<bool>(f)`;
  - the stored callable's type: `f.target_type()` (`std::type_info`) -- a
    lambda's is a compiler-generated closure name; a plain function
    pointer's is the pointer type;
  - a plain function pointer's address: `f.target<R(*)(Args...)>()` when the
    target is one -- a symbol name is then possible (`dladdr`, or a
    registry).  Lambdas and functors cannot be seen into.

## Design questions (decide before building)

1. **How a `std::function` relates to `mt_function`:** its own metatype (a
   callable wrapper) whose info points at an `mt_function` signature; or
   `mt_function` with a wrapper flavour; or something else.  Every
   reflection consumer switching on metatype is affected -- survey them
   (`grep -rn 'mt_function\|is_function()' xo-*/src xo-*/include`).
2. **What a value prints:** set/empty only; plus the target's type name;
   plus a function pointer's address / symbol.
3. **`noexcept`, `std::move_only_function`, `std::function_ref` (C++26):**
   in scope now, or later.
4. **Canonical name:** `type_name<T>` already spells `std::function<..>`;
   whether the reflected signature should be exposed alongside it, and how
   printjson writes it (e.g. `_signature_`, or only through the declared
   type).

## Not here

- `std::regex` (DynamicEndpoint's `uri_regex_`, summarised as `"N
  captures"`) is not a callable: its own decision, recorded in
  `.xo-backlog/xo-websock/issues/16`.

**Done when:** a `std::function` member reflects with its signature, prints
through printjson's generic path with no per-struct printer, and
DynamicEndpoint's three function members do (`xo-websock/issues/16`).
