# 01 — reflect `T*` the way `rp<Object>` is reflected

Status: done 2026-09-21
Type: feature

`xo-reflect` describes `xo::ref::rp<Object>` as a pointer — `mt_pointer`, 0 or 1
children — via `EstablishTdx<rp<Object>>` and `RefPointerTdx`. A raw `T*` gets
nothing: `EstablishTdx`'s primary template returns `AtomicTdx::make()`, so a raw
pointer reflects as an opaque atom with no children.

Add `EstablishTdx<T*>` + `RawPointerTdx`, which is `RefPointerTdx` with
`ptr->get()` replaced by `*ptr`.

## What prompted it

`FlywheelInfo::strong_` is a `const DHandleArena<ObjectSlot> *` — a borrowed
pointer to a live root set, which is what retired `RootSetInfo` (see the
provenance section). Rendering it needed `JsonPrinter_RootSet` keyed on the
POINTER type, doing its own dereference:

```cpp
const RootSet ** pp = this->check_recover_native<const RootSet *>(tp, p_os);
const RootSet * rs = *pp;
```

With `T*` reflected, the printer would key on the pointee and
`print_generic_pointer` (`xo-printjson/src/printjson/PrintJson.cpp:61`) would do
that hop.

**The payoff for that one member is a single dereference.** The case for doing
this is the general shape — a view model holding a borrowed pointer — which is
expected to recur as `FlywheelInfo` shrinks. Do not let this ticket's length
suggest otherwise.

## Design decisions (author, 2026-09-21)

1. **`Reflect::require<void>()` will be made to work**, by whatever
   specialisations that needs. `void*` is therefore not an obstacle.
2. **Dangling pointers are not reflection's problem.** Traversal assumes valid
   state exactly as every other operation does; the caller signed up for that by
   using C++. Reflection gets no special conservatism here.
3. **Reflection is a CURATED view, not a mirror of layout.** `rp<Foo>` has a
   reference count and reflection omits it. Addresses and byte pointers are
   grey-area on the same axis — worth surfacing, never forcing.
4. **`const char*` gets its own registration**, treated as a string rather than
   a pointer.

Decision 3 is the one that shrinks the rest of this ticket, and it does so
because curation already has a mechanism: `StructReflector` names members
explicitly, which is why `MemorySizeInfo::detail_` is absent from
`reflect_flywheel_info`. So `EstablishTdx<T*>` never decides WHETHER a raw
pointer is exposed — only what happens to one a human already named. Exposure
stays a per-member choice one level up.

## Mechanical requirements

These are not taste; getting them wrong is a silent wrong answer.

### Establish the child as `std::remove_cv_t<T>`

`typeid` strips top-level cv, and `TypeDescrBase::require` looks up by
`TypeInfoRef` before anything else (`xo-reflect/src/reflect/TypeDescr.cpp:75`),
so `require<const Foo>()` and `require<Foo>()` land on the SAME `TypeDescr`. But
the `canonical_name` argument comes from `type_name<T>()`, so whichever call
arrives first names the shared slot — and a struct's json `_name_` is drawn from
that name. Reflecting a `const T*` member could therefore leave a type reading
`"Foo const"`, depending on link order.

`remove_cv_t<T>` for the child makes it order-independent. `child_tp` then needs
a `const_cast`, since `TaggedPtr` is `void*`-based. `RefPointerTdx` never meets
this: `rp<Object>::get()` returns a non-const `Object*`.

Was **unverified** when this ticket was written — reasoned from [expr.typeid]/5
plus the lookup order above, not observed. Measured on implementation; see the
done-when section. It held, and `pointer-to-const-shares-one-pointee` in
`xo-reflect/utest/PointerTdx.test.cpp` now pins it.

Note pointer types themselves stay distinct — `typeid(const Foo*)` is NOT
`typeid(Foo*)`, since there the const is not top-level — so pointer-to-const and
pointer-to-mutable keep separate descriptors while sharing one pointee. That is
the wanted behaviour.

### `void*` reports 0 children even when non-null

Decision 1 makes `EstablishTdx<void*>` instantiate cleanly, but a void pointee
still has nothing to traverse into, so `child_tp` should never be asked to
invent a `TaggedPtr` to void.

`MemorySizeInfo::lo_`/`hi_` (`const void *`, reflected today) are unaffected
either way: `PrintJson::print_aux` consults `printer_map_` BEFORE the metatype
switch (`PrintJson.cpp:144`), so `JsonPrinter_address` keeps winning regardless
of what metatype the type carries.

### `print_generic_pointer` should emit `null`, not `{}`

It currently prints `{}` for a null pointer (`PrintJson.cpp:73`),
distinguishable from a real struct only by the absent `_name_`.
`JsonPrinter_RootSet` already emits `null`. Without this, routing `strong_`
through the generic path is a small regression on today's wire output.

### `const char*` as a string

`JsonPrinter_string` (`PrintJson.cpp:362`) is the model but takes a
`std::string`, so it sidesteps two things a `const char*` printer owns:

- **null** — render `null`, not `""`, and not a crash
- **escaping** — inherits `.xo-backlog/xo-printjson/issues/04`: `quot()` escapes
  via ppsink (backslash-x00) where json requires backslash-u0000

A full specialisation out-ranks the partial one, so `EstablishTdx<const char*>`
returning `AtomicTdx::make()` reads as "exempt from `T*`".

## What landed

`RawPointerTdx<T>` in `xo-reflect/include/xo/reflect/pointer/PointerTdx.hpp`,
beside `RefPointerTdx`, plus `EstablishTdx<T *>` in `Reflect.hpp` and full
specialisations exempting `char *` / `const char *`. `print_generic_pointer`
now emits `null`. `JsonPrinter_RootSet` was re-keyed from the pointer onto the
POINTEE, which is this ticket's concrete payoff on existing code -- and the
byte-exact frame test in `xo-object2/utest/flywheel_frame.test.cpp` passes
unchanged, which is far better evidence than a synthetic test.

Tests: six cases in `xo-reflect/utest/PointerTdx.test.cpp` (new file), three in
`xo-printjson/utest/PrintJson.test.cpp`, both tagged `[rawpointer]`.

### Falsified, three ways

| reverted | breaks |
|---|---|
| `EstablishTdx<T*>::make()` -> `AtomicTdx` | 4 reflect, 2 printjson, **4 object2 flywheel** |
| `const char *` exemption removed | 1 reflect (metatype only -- see below) |
| `print_generic_pointer` back to `{}` | 3 printjson |

The flywheel row is the useful one: the frame breaking proves the re-keyed
printer genuinely reaches the store through raw-pointer reflection.

## Four things this ticket predicted wrongly

Recorded per CONVENTIONS rule 6; each was plausible and each cost time.

**1. `Reflect::require<void>()` already worked.** Decision 1 anticipated
needing new specialisations; none were required.

```
require<void>()  = 0x3ef90dd0  name=void
```

So `void*` needed only `RawPointerTdx`'s `if constexpr (std::is_void_v<T>)`
guard returning 0 children, not groundwork.

**2. The `const char *` string printer already existed.**
`provide_string_printer<char *>` and `<char const *>` have been registered in
`PrintJson::provide_std_printers` since long before this ticket
(`xo-printjson/src/printjson/PrintJson.cpp:726`). What was actually missing was
the null case, and it did not merely render badly -- **it segfaulted**:

```
about to print a null const char*...
Segmentation fault (core dumped)
```

Pre-existing and reachable without any of this work; it simply had no test.
Fixed with an `if constexpr (std::is_pointer_v<T>)` branch in
`JsonPrinter_string`.

**3. The exemption protects the METATYPE, not the wire.** Removing it breaks
only `char-pointers-are-exempt`; rendering is unaffected, because `print_aux`
consults `printer_map_` before the metatype switch, so the string printer wins
either way. The exemption is still right -- a C string is text, not a container
of one char -- but it is not what keeps strings rendering as strings, and the
ticket implied it was.

**4. `{}` -> `null` reached further than "a small regression on today's wire".**
Two existing cases in `xo-printjson/utest/FopJson.test.cpp` pinned `{}` for an
EMPTY FOP, which takes the same generic path. Both updated, and the change is
an improvement rather than a cost: an `ObjectSlot` is an erased fop and already
rendered `null` when empty, so the tree had two spellings for one condition.
`print-obj-renders-an-empty-fop-as-braces` is renamed `...-as-null`.

## A new constraint this introduced

**`Reflect::require<T *>()` now requires `T` to be complete.** It did not
before -- a pointer to an incomplete type fell to the primary template and
reflected as an atom.

```bash
# struct Incomplete;  Reflect::require<Incomplete *>();
#   EstablishTypeDescr.hpp:42: invalid use of incomplete type 'struct Incomplete'
#   TypeDescr.hpp:109: invalid application of 'sizeof' to incomplete type
```

Note `sizeof` at `TypeDescr.hpp:109`: `require<T>()` needs completeness whatever
Tdx it ends up with, so this is `EstablishTdx<T*>::make()` calling
`Reflect::require<remove_cv_t<T>>()` inheriting an existing requirement, not a
new one of its own. Same bargain `RefPointerTdx` makes for `rp<Object>`.

Nothing in the tree hits it -- the only reflected raw pointers are
`MemorySizeInfo::lo_`/`hi_` (`const void *`, and `void` is fine). But merely
DECLARING such a member is still free; only reflecting it is not.

## Superseded one day later — the flywheel payoff, not the feature

`JsonPrinter_RootSet` was re-keyed onto the pointee here, reached through
`FlywheelInfo::strong_` via `print_generic_pointer`. On 2026-09-22
`FlywheelInfo` was retired (`.xo-backlog/xo-facet/issues/02`) and
`JsonPrinter_AllocFlywheel` now reaches the store DIRECTLY through
`AllocFlywheel::strong_root_set()`.

So the flywheel frame no longer exercises raw-pointer reflection. The pointee
key is still right, and everything else here stands -- raw pointers reflect, a
null `const char*` no longer segfaults, `{}` became `null`. But this ticket's
"concrete payoff on existing code" lasted a day, and the coverage that matters
is the `[rawpointer]` cases in `xo-reflect/utest/PointerTdx.test.cpp`, not the
byte-exact frame.

Worth knowing before citing the frame as evidence that raw pointers work.

## Curation choices, deliberately left open

Per decision 3 — defaults are fine until someone cares.

| | |
|---|---|
| `char *` | presumably goes with `const char *` |
| `std::byte *`, `unsigned char *` | memory, not text. Live: `DArena::Checkpoint::free_`. Under blanket `T*` they become pointer-to-atom: useless but harmless. If they should read as addresses that is a PRINTER registration reusing `JsonPrinter_address`, not a Tdx specialisation — `const void *` is already handled that way, and keeping the two mechanisms separate is the point |

## Verified: function pointers keep their own specialisation

`EstablishTdx<Retval (*)(Args...)>` already exists
(`xo-reflect/include/xo/reflect/Reflect.hpp:75`). A `T*` partial specialisation
does not capture it — partial ordering prefers the function-pointer one. This is
the load-bearing safety claim, so it was run rather than assumed:

```cpp
// g++ -std=c++20
template <typename T>                struct E              { static const char * who() { return "primary"; } };
template <typename R, typename... A> struct E<R(*)(A...)>  { static const char * who() { return "fnptr"; } };
template <typename T>                struct E<T*>          { static const char * who() { return "T*"; } };
// int* -> T*   const void* -> T*   const char* -> T*   S* -> T*   int(*)(double) -> fnptr
```

## Blast radius

Small, because almost nothing in the tree is reflected as a struct yet. The
whole set of reflected members:

```bash
grep -rn "\.reflect_member(\"" --include=*.cpp xo-*/src/
```

As of 2026-09-21 that is nine lines in one file,
`xo-object2/src/object2/reflect_flywheel_info.cpp`: `MemorySizeInfo` (7 members)
and `FlywheelInfo` (2). The only raw-pointer members among them are
`MemorySizeInfo::lo_`/`hi_` (`const void *`, printer-intercepted) and
`FlywheelInfo::strong_`.

Restricted to `src/` deliberately. A wider grep also matches
`StructReflector.hpp`'s own declaration and doc comment, the `REFLECT_MEMBER`
macros, and `xo-reflect/utest/StructReflector.test.cpp` — none of which are
registrations, so the wider form does not establish this count.

## A wrong argument, recorded so it is not re-made

While designing this I argued that following a raw pointer is a LIFETIME claim
reflection cannot make, and cited `MemorySizeInfo::detail_` — omitted from
`reflect_flywheel_info` because it points into the stack frame of whoever ran
the visit — as precedent for a conservative default.

That over-reached. `detail_`'s omission is a fact about one member whose pointee
was a visitor's stack frame, not evidence that raw pointers need a conservative
default in general. And it proves the opposite of what it was cited for: the
member was excluded by a HUMAN at `reflect_member`, which is precisely the
curation mechanism that makes a permissive `EstablishTdx<T*>` safe.

Plausible because `rp<T>` really does carry a lifetime guarantee that `T*` does
not. The error was treating that difference as reflection's problem rather than
the caller's — see decision 2.

The same correction applies to a cycles argument made alongside it: that raw
back/parent pointers would make `.xo-backlog/xo-printjson/issues/02` (cyclic
object graphs) load-bearing rather than deferred. Weaker than claimed for the
same reason — a cycle only arises if someone NAMES a member that closes a loop,
which is a visible local decision, not something the specialisation inflicts on
the tree.

## Done when — all met 2026-09-21

- [x] `Reflect::require<Foo*>()->metatype() == Metatype::mt_pointer`, with
  `n_child` 0 for null and 1 otherwise
- [x] a reflected `const Foo *` member renders as the pointee, and the
  pointee's canonical name does not depend on establishment order
- [x] `Reflect::require<const char *>()` still reflects as an atom, and renders
  as a quoted string including the null case
- [x] `JsonPrinter_RootSet` re-keyed on `DHandleArena<ObjectSlot>`; frame
  byte-identical, and unchanged through the python binding too
- [x] `xo-build --sweep` ok in both stages (71 attempted: 43 ok, 0 failed),
  umbrella ctest 44/44

```bash
.build/xo-reflect/utest/utest.reflect    "[rawpointer]"
.build/xo-printjson/utest/utest.printjson "[rawpointer]"
.build/xo-object2/utest/utest.object2     "[flywheel]"
```

The `const Foo*` claim rested on typeid stripping top-level cv, which this
ticket marked unverified. Now measured:

```
require<Foo>       = 0x3ef8fce0  name=Foo
require<const Foo> = 0x3ef8fce0  name=Foo      <- same TypeDescr
typeid(Foo*)==typeid(const Foo*): NO           <- pointers stay distinct
```

## Provenance

Arose 2026-09-21 while retiring `RootSetInfo` from `xo-facet/FlywheelInfo.hpp`.
That struct copied five fields out of `DHandleStore` and was described a third
time by a `StructReflector`; it was replaced by `visit_object_slots` /
`visit_free_list` on the store plus `JsonPrinter_RootSet`, with
`FlywheelInfo::strong_` becoming the borrowed pointer that raised this question.
