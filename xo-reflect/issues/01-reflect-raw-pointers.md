# 01 — reflect `T*` the way `rp<Object>` is reflected

Status: open (design settled 2026-09-21, unimplemented)
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

**Unverified in this tree** — reasoned from [expr.typeid]/5 plus the lookup
order above, not observed. Worth confirming before relying on it:

```cpp
/* in a utest: do these produce one TypeDescr or two? */
REQUIRE(Reflect::require<const MemorySizeInfo>() == Reflect::require<MemorySizeInfo>());
```

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

## Done when

- `Reflect::require<Foo*>()->metatype() == Metatype::mt_pointer`, with
  `n_child` 0 for null and 1 otherwise
- a reflected `const Foo *` member renders as the pointee, and the pointee's
  canonical name does not depend on establishment order
- `Reflect::require<const char *>()` still reflects as an atom, and renders as a
  quoted string including the null case
- `JsonPrinter_RootSet` can be re-keyed on `DHandleArena<ObjectSlot>` and the
  frame is byte-identical apart from that — the existing byte-exact test in
  `xo-object2/utest/flywheel_frame.test.cpp` is the check
- `xo-build --sweep` ok in both stages

## Provenance

Arose 2026-09-21 while retiring `RootSetInfo` from `xo-facet/FlywheelInfo.hpp`.
That struct copied five fields out of `DHandleStore` and was described a third
time by a `StructReflector`; it was replaced by `visit_object_slots` /
`visit_free_list` on the store plus `JsonPrinter_RootSet`, with
`FlywheelInfo::strong_` becoming the borrowed pointer that raised this question.
