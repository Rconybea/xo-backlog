# 02 — reflect `DArenaVector<T>` as a vector

Status: done 2026-09-21
Type: feature

`xo::mm::DArenaVector<T>` has no `EstablishTdx` specialisation, so it falls to
the primary template and reflects as an opaque atom:

```cpp
static std::unique_ptr<TypeDescrExtra> make() { return AtomicTdx::make(); }
```

`mt_atomic`, no children. In printjson that reaches the default case of
`PrintJson::print_aux` and emits
`<error-json-printer-not-found ... metatype=mt_atomic>`.

Add `EstablishTdx<mm::DArenaVector<T>>`, alongside the existing
`std::vector` / `std::array` / `std::pair` specialisations.

## Nothing reflects one today

Which is why this has not bitten:

```bash
grep -rn "DArenaVector" --include=*.hpp --include=*.cpp \
     xo-reflect/ xo-reflectutil/ xo-reflectable2/ xo-printjson/
grep -rn "require<.*DArenaVector\|reflect_member.*DArenaVector" \
     --include=*.cpp --include=*.hpp xo-*/
```

Both empty as of 2026-09-21. The one `DArenaVector` anywhere near the wire is
`DHandleStore::strong_refs_`, which is private and reached through
`visit_object_slots` rather than reflection (see issue 01's provenance).

## Verified: `StlVectorTdx` already handles it

`StlVectorTdx<VectorT>` (`xo-reflect/include/xo/reflect/vector/VectorTdx.hpp:42`)
is generic over the container, not tied to `std::vector`. It documents its
requirements as `value_type`, `size()`, and `operator[]` yielding an lvalue;
`DArenaVector` has all three (`xo-arena/include/xo/arena/DArenaVector.hpp:29`,
`:68`, `:83`).

That is the load-bearing claim — "this is one specialisation, not a new Tdx
class" — so it was run, against the umbrella build, without touching the repo:

```cpp
/* scratch TU; links -lreflect -lxo_arena -lxo_indentlog2 -lxo_ppsink -lrefcnt */
auto tdx = StlVectorTdx<DArenaVector<double>>::make();

auto v = DArenaVector<double>::map(ArenaConfig()
                                   .with_name(ArenaNameStr::from_cstr("scratch"))
                                   .with_size(64*1024));
v.push_back(1.5); v.push_back(2.5); v.push_back(3.5);

tdx->n_child(&v);                                  //  3
tdx->n_child_fixed();                              //  0
tdx->has_contiguous_storage();                     //  1
tdx->fixed_child_td(0)->canonical_name();          //  "double"
tdx->child_tp(1, &v).td()->canonical_name();       //  "double"
*tdx->child_tp(1, &v).recover_native<double>();    //  2.5
```

So the change is:

```cpp
template <typename T>
std::unique_ptr<TypeDescrExtra> EstablishTdx<mm::DArenaVector<T>>::make() {
    Reflect::require<T>();
    return StlVectorTdx<mm::DArenaVector<T>>::make();
}
```

plus the forward declaration beside the other `EstablishTdx` specialisations
(`xo-reflect/include/xo/reflect/Reflect.hpp:57` is the `std::vector` one).

Expected to need no printjson change at all, since `print_generic_vector`
handles `mt_vector`. **Confirmed on implementation** — see below.

## Home: xo-reflect, but it needs a declared dependency

Levelization allows it, which is the opposite of what the arena's position
suggests:

```bash
xo-deps --why=xo-reflect:xo-arena -q   # xo-reflect -> xo-indentlog2 -> xo-arena
xo-deps --why=xo-arena:xo-reflect -q   # nothing, exit 1
```

So the specialisation cannot live in xo-arena (it would invert the graph) and
xo-reflect is the natural home.

But xo-reflect does not DECLARE xo-arena:

```bash
grep -n xo_dependency xo-reflect/src/reflect/CMakeLists.txt
#   refcnt, xo_ppsink, xo_indentlog2
```

while already including an arena header:

```bash
grep -rn "xo/arena/" xo-reflect/include/ xo-reflect/src/
#   xo-reflect/include/xo/reflect/cx/ReflectAppcx.hpp:11: <xo/arena/MemorySizeInfo.hpp>
```

That is a pre-existing reliance on the transitive path through xo-indentlog2,
not something this ticket introduces — but including `DArenaVector.hpp` deepens
it, and declaring `xo_dependency(${SELF_LIB} xo_arena)` is the honest fix.
Relevant machinery: `.xo-backlog/generated-find-dependency`, which generates
`find_dependency()` from the `xo_deps` property, so an undeclared dep is also a
missing `find_dependency` in the installed Config.cmake.

## What landed

`xo-reflect/include/xo/reflect/Reflect.hpp` gains the declaration beside the
other container specialisations and the definition beside `std::vector`'s, plus
a FORWARD DECLARATION of `xo::mm::DArenaVector` rather than an include:

```cpp
namespace mm { template <typename T> struct DArenaVector; }
```

`EstablishTdx`'s specialisation needs only the name declared; `make()` is a
template, so `DArenaVector` has to be complete only where someone calls
`Reflect::require<DArenaVector<T>>()`, and there they have necessarily included
the real header. Including `xo/arena/DArenaVector.hpp` from `Reflect.hpp` would
put it on the include path of every TU that reflects anything.

`xo_dependency(${SELF_LIB} xo_arena)` added as planned. The installed
Config.cmake picked it up with no further work — xo-reflect is already on the
generated path:

```bash
grep -n find_dependency ~/local/lib/cmake/reflect/reflectConfig.cmake
#   refcnt, xo_ppsink, xo_indentlog2, xo_arena, subsys
```

Tests: three cases in `xo-reflect/utest/VectorTdx.test.cpp` (`[darenavector]`)
covering empty, two elements with address and value recovery, and a STRUCT
element to pin that `Reflect::require<Element>()` runs; two in
`xo-printjson/utest/PrintJson.test.cpp` (same tag) pinning `[1.5, 2.25, -3]`
and `[]`.

### Falsified

Reverting `make()` to `AtomicTdx::make()` — the pre-ticket behaviour — fails
all five, and printjson reproduces exactly the error this ticket opened with:

```
<error-json-printer-not-found :type xo::mm::DArenaVector<double> :metatype mt_atomic>
```

So the printjson cases are load-bearing rather than incidental: they are what
shows that describing the container was the whole of the work.

Verified with `xo-build --sweep` (71 attempted: 43 ok, 0 failed, both stages)
and umbrella ctest 44/44.

## Incidental findings, not required by this ticket

**`StdVectorTdx<Element>` duplicates `StlVectorTdx<std::vector<Element>>`.**
Body for body — same `has_contiguous_storage`, same `n_child`, same
`n_child_fixed`, same `child_tp`, and `fixed_child_td` differing only in
spelling `Element` vs `typename VectorT::value_type`
(`VectorTdx.hpp:42` and `:89`). `StdVectorTdx` derives from `VectorTdx`
directly rather than from `StlVectorTdx`, so the two are unrelated by
inheritance. Routing `DArenaVector` through `StlVectorTdx` makes the redundancy
visible; collapsing `std::vector` onto it is a separate, optional tidy.

**`has_contiguous_storage()` has no consumers.**

```bash
grep -rn "has_contiguous_storage" --include=*.cpp --include=*.hpp xo-*/ \
  | grep -v "VectorTdx.hpp"    # empty
```

Pure abstract surface that nothing calls, so whatever a new specialisation
answers is unobservable today.

**`n_child` narrows `size()` to `uint32_t`.** Implicit, in both
`StlVectorTdx` and `StdVectorTdx`. Pre-existing and shared with `std::vector`,
so not a reason to treat `DArenaVector` differently; noted because a
`DArenaVector`'s capacity is fixed at construction, which bounds it by the
arena rather than by the type.

**`child_tp` reaches elements through `operator[]` -> `_address_of`.** That
function was one of the nine sites in `.xo-backlog/xo-arena/issues/05` — element
0 used to be the arena's back pointer, and writing it destroyed the preamble.
Fixed, but reflection would become a second consumer of it, so a regression
there would surface here too.

## Done when — all met 2026-09-21

- [x] `Reflect::require<DArenaVector<double>>()->metatype() == Metatype::mt_vector`
- [x] a populated `DArenaVector` renders as a json array, with element count and
  values matching, and an empty one as `[]`
- [x] `Reflect::require<DArenaVector<Foo>>()` establishes `Foo` as the element
  type for a struct element, not just a scalar
- [x] xo-reflect declares xo_arena, and the installed `reflectConfig.cmake`
  carries the matching `find_dependency`
- [x] `xo-build --sweep` ok in both stages

```bash
.build/xo-reflect/utest/utest.reflect "[darenavector]"
.build/xo-printjson/utest/utest.printjson "[darenavector]"
```

The incidental findings below were NOT acted on — they remain open as described.

## Provenance

Asked 2026-09-21, immediately after issue 01, while looking at what else in the
arena container family reflection does not describe. Neighbours not covered
here: `DArenaHashMap`, `DCircularBuffer`.
