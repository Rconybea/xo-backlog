# 06 -- reflect enums: EnumReflector, REFLECT_ENUM, EnumTdx

Status: open
Type: feature
Milestone: reflection-driven-json

An enum is an opaque leaf to xo-reflect today. `EstablishTdx<T>`'s primary
template gives every `T` an `AtomicTdx`
(`xo-reflect/include/xo/reflect/Reflect.hpp`), which has no names and no
values. So each enum that is printed has a hand-written `*_descr()`
(`runstate_descr`, `endpoint_kind_descr`, `tokentype_descr`, ..: about 130
enums in the tree, 29 prettifier adaptations of descr functions). The
websock printers summarise `DynamicEndpoint::kind_` (EndpointKind) and
`WebserverImpl::state_` (Runstate) with `member_as`, although both are
plain enum members.

Split out of `issues/04` (RC, 2026-10-05).

## Design

- **`EnumReflector<E>`**, mirroring `StructReflector`:
  - its constructor establishes the `TypeDescr`;
  - `reflect_enumerator(name, value)` collects enumerators;
  - `require_complete()`, also called by the destructor, installs an
    `EnumTdx` with `assign_tdextra`, once, guarded by a per-type static
    flag (`is_incomplete()`).
- **Macros**, mirroring the struct ones:
  - `REFLECT_ENUM(er, stopped)` expands to
    `er.reflect_enumerator("stopped", decltype(er)::enum_t::stopped)`, for
    scoped and unscoped enums alike;
  - `REFLECT_EXPLICIT_ENUM(er, "name", value)`.
- **`EnumTdx`**, type-erased: enumerators in declaration order, as
  `{int64 value, name}`. It provides:
  - `n_enumerator()`, `enumerator_name(i)`, `enumerator_value(i)`;
  - `name_of(void const *)`: the name, or none for a value with no
    enumerator;
  - `assign_from_name(name, void *)`.

  It reads and writes the object through small functions captured when it
  is built, which know the underlying type, so `EnumTdx` itself is not a
  template.
- **Metatype stays `mt_atomic`** (RC): an enum composes no other type,
  like string, int, float and bool, which are all atomic. The new
  capability is a hook `TypeDescrExtra::enum_info()`, `nullptr` by
  default, like `fn_info()`, plus `TypeDescr::is_enum()`, since there are
  many enum types.
- **printjson** prints a reflected enum value as its name, a json string
  (`"stopped"`). A value with no enumerator prints as its integer, a json
  number (RC), so a consumer can tell the two apart. Flag enums are out of
  scope. `JsonMembers::printable` accepts a reflected enum.
- **Where an enum is reflected:** an enum has no member functions, so a
  companion does it, e.g. `RunstateUtil::reflect_self(table)`, and
  websock's reflect list calls it.
- **The existing `*_descr()` functions stay** (RC). Re-deriving them from
  reflection, for one source of truth, would be a separate, later pass.

## Path

1. xo-reflect: `EnumReflector`, `EnumTdx`, `enum_info()` / `is_enum()`,
   the macros, and tests: names, round trip, an unreflected value, scoped
   and unscoped. No consumers yet.
2. printjson: print reflected enums, with tests (an enum member prints its
   name; an unknown value prints its integer).
3. websock: reflect `EndpointKind` and `Runstate`; `kind_` and `state_`
   become reflected members. The golden snapshot diff shows the result.

**Done when:** `kind_` and `state_` print through reflection, with no
`member_as`, and xo-reflect has tests for the cases in step 1.
