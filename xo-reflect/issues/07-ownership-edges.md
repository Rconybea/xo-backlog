# 07 -- ownership edges

Status: open
Type: feature
Milestone: reflection-driven-json

Describe, for each edge from a reflected object to another, whether the
holder owns what it reaches.  A traversal that takes locks
(`xo-reflect/issues/08`, `xo-printjson/issues/09`) may descend only through
owning edges; through any other edge it may name the target, but must not
read it.  Print placement (`xo-printjson/issues/08`) follows the same edges.

## Design (RC, 2026-10-08)

Three kinds:

| kind | default for | a traversal |
|---|---|---|
| `owning` | by-value members, container elements, `std::unique_ptr` | descends; places the target in full |
| `shared` | `rp<T>` | descends at first appearance; a ref afterwards |
| `borrowed` | `T*` | never dereferences; a ref only |

Why `rp<T>` defaults to shared-and-placed (RC): any holder of an `rp<T>`
may become T's last owner, so if T needs synchronizing, its mutex lives in
T, not in a holder.  A holder's lock guards only the `rp` slot.  So every
holder's path acquires holder-then-T, consistently, and the slot keeps T
alive while a traversal is inside the holder.  The discipline this relies
on, to document rather than enforce: a self-synchronized T never calls into
a holder while holding its own lock.

Override per member, on the builder `reflect_member` returns:

```cpp
REFLECT_MEMBER(sr, sender).owning();
REFLECT_MEMBER(sr, pjson).borrowed();   // e.g. a back-reference, or "placed elsewhere"
```

## Changes

- `enum class Ownership { owning, shared, borrowed }`.
- `PointerTdx::pointee_ownership()`.  `unique_ptr` and `rp` share
  `RefPointerTdx` today (`include/xo/reflect/pointer/PointerTdx.hpp:46`), so
  `RefPointerTdx::make()` takes the kind; each `EstablishTdx` specialization
  passes its own (`include/xo/reflect/Reflect.hpp:59` rp, `:92` `T*`,
  `:105` unique_ptr).
- `StructMember` (`include/xo/reflect/struct/StructMember.hpp:185`) gains an
  optional override; `StructMember::ownership()` resolves override, else the
  member type's default, else `owning` for a non-pointer.
- `StructReflector::reflect_member` (`include/xo/reflect/StructReflector.hpp:57`)
  returns a builder instead of `void`; `REFLECT_MEMBER` /
  `REFLECT_LITERAL_MEMBER` / `REFLECT_EXPLICIT_MEMBER` (`:146-167`) expand to
  that call, so existing uses compile unchanged.  `xo-reflect/issues/08`
  adds `.guarded_by()` to the same builder.
- Container elements take the element type's default; no per-element
  override until a case needs one.

```bash
grep -n "class RefPointerTdx\|class RawPointerTdx" xo-reflect/include/xo/reflect/pointer/PointerTdx.hpp
grep -n "class EstablishTdx<" xo-reflect/include/xo/reflect/Reflect.hpp
grep -n "void reflect_member\|#define REFLECT" xo-reflect/include/xo/reflect/StructReflector.hpp
```

**Done when:** each pointer type reports its default ownership, a member's
resolved ownership honours an override, and every existing `REFLECT_*`
use builds unchanged (`xo-build --sweep -q`).
