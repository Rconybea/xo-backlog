# 02 — retire the flywheel's Info classes

Status: done 2026-09-22
Type: refactor

A flywheel frame used to be built by copying the flywheel into a tree of view
structs, reflecting those, and letting `print_generic_struct` render them. All
four are now gone; a frame is produced by three bespoke printers reading the
live flywheel.

| retired | was | replaced by | when |
|---|---|---|---|
| `PoolInfo` | copy of `MemorySizeInfo` | reflecting `MemorySizeInfo` directly | 2026-09-15 |
| `SlotInfo` | shadow of an erased fop | `ObjectSlot` + `JsonPrinter_ObjectSlot` | 2026-09-20 |
| `RootSetInfo` | copy of the root set | `visit_object_slots` / `visit_free_list` + `JsonPrinter_RootSet` | 2026-09-21 |
| `FlywheelInfo` | frame envelope | `JsonPrinter_AllocFlywheel` | 2026-09-22 |

`AllocFlywheel::snapshot()` and `DHandleStore::snapshot()` went with them, as
did `xo::facet::reflect_flywheel_info()` and its subsystem-crossing home in
xo-object2.

## Why they kept looking necessary

Each layer had a real reason while it stood, and the reason expired without
anything noticing:

- `PoolInfo` existed because `MemorySizeInfo::detail_` must not be followed
  from a snapshot. But StructReflector names members explicitly, so the
  pointer can simply be left unnamed -- the copy was solving a problem
  omission already solved.
- `SlotInfo` existed because an erased fop cannot describe itself: `FopTdx`'s
  erased path rotates through `AReflectable` and throws for a representation
  that has not opted in. That stopped being true when `ObjectSlot` was made to
  DERIVE from `obj<ATop>` rather than alias it, so it no longer matches
  `EstablishTdx<obj<AFacet,DRepr>>` and reflects as an atom a bespoke printer
  can key on.
- `RootSetInfo` existed to carry a copy of a private container. A visitor
  hands out the same information without exposing it, and without materialising
  anything.
- `FlywheelInfo` existed as the envelope -- the place a frame sequence number
  would go. A printer is also such a place.

**The common shape: each described a thing that could already describe itself,
and each cost a second place to keep in step.**

## The rule that came out of it

**Reflection describes STORED members. A view model with COMPUTED fields needs
a printer.**

That is what decided the last one, and it is the durable part. `size`,
`capacity` and `live` are not members of anything -- they are
`strong_refs_.size()`, `.capacity()`, and `size - freelist.size()`. A
`StructReflector<DHandleStore>` was explored on 2026-09-22 and rejected for
exactly this: it would have rendered `slots` and `free` correctly (verified --
a `DArenaVector<ObjectSlot>` routes its elements to `JsonPrinter_ObjectSlot`)
while losing `capacity` entirely, since no consumer can rederive it from the
wire.

Two options were weighed for recovering it and both rejected by the author:

- a bespoke `DArenaVector<T>` printer carrying `capacity` -- would make a
  vector render as a json OBJECT rather than an array, everywhere
- widening `VectorTdx` with a `capacity()` member -- pushes a `DArenaVector`
  concern onto every vector, including `std::vector` where capacity is an
  implementation detail rather than semantics

Recorded because the second looks obviously right until you notice it changes
`std::vector`'s wire to answer a question only one container is asking.

## Access, and the friend that was not needed

`DHandleStore`'s members are private, and `StructReflector::reflect_member`
takes a pointer-to-member, so reflecting it would have needed access. The
pattern settled on was a template forward-declared in xo-facet, befriended, and
DEFINED in a subsystem that can reach a `StructReflector` -- preserving
levelization at the cost of declaring a template xo-facet never defines.

Verified to work before the approach was dropped:

```cpp
namespace facet {
    template <typename Store> struct ReflectHandleStore;    // defined above us

    template <typename Storage, typename Handle>
    class DHandleStore {
        template <typename Store> friend struct ReflectHandleStore;
        ...
```

A `&Store::private_member_` taken from the friend compiles and works. Kept here
because the pattern is sound and will be wanted the next time something below
the reflection layer needs describing -- it just was not worth its cost here.

## Where things ended up

`JsonPrinter_AllocFlywheel`, `JsonPrinter_RootSet` and `JsonPrinter_ObjectSlot`
all live in `xo-printjson/src/printjson/PrintJson.cpp`, installed by one
`provide_flywheel_printers()`, which ALSO reflects `MemorySizeInfo`.

That last part is the real ergonomic win. The registration used to be
`xo::facet::reflect_flywheel_info()` in xo-object2 -- homeless, because
xo-facet owns `AllocFlywheel` but cannot reach a `StructReflector`
(`xo-printjson -> xo-printable2 -> xo-facet`), and every caller had to remember
it. Its own doc said it was expected to move to wherever the consumer landed.
The consumer is the printer, so it moved there, and constructing a `PrintJson`
is now sufficient.

`AllocFlywheel` gained `strong_root_set()`, which hands out no more than
`FlywheelInfo::strong_` already did.

xo-printjson now DECLARES `xo_facet` and `xo_arena`; it had been including
both while reaching them only transitively.

## Wire change

One key, in the envelope:

```
"_name_": "FlywheelInfo"   ->   "_name_": "Flywheel"
```

Everything else is byte-identical, which the existing byte-exact test in
`xo-object2/utest/flywheel_frame.test.cpp` establishes -- it needed exactly that
one edit.

## This superseded part of xo-reflect/01

Raw-pointer reflection landed on 2026-09-21 with `JsonPrinter_RootSet` re-keyed
onto the pointee, reached through `FlywheelInfo::strong_` (a
`const HandleStore *`) via `print_generic_pointer`. A day later `FlywheelInfo`
was gone and `JsonPrinter_AllocFlywheel` reaches the store DIRECTLY through
`strong_root_set()`.

So **the flywheel frame no longer exercises raw-pointer reflection at all.**
The pointee key is still correct, and xo-reflect/01 stands on its own (raw
pointers reflect; a null `const char*` no longer segfaults; `{}` became
`null`) -- but its stated payoff on existing code lasted one day. Noted so the
`[rawpointer]` tests in `xo-reflect/utest/PointerTdx.test.cpp` are understood
as the coverage, rather than the frame being assumed to cover it.

## Verification

```bash
.build/xo-object2/utest/utest.object2 "[flywheel]"
.build/xo-facet/utest/utest.facet     "[snapshot]"
```

Falsified twice, both compiling cleanly so the binary is not stale:

| reverted | result |
|---|---|
| `AllocFlywheel` printer not registered | `<error-json-printer-not-found :type xo::facet::AllocFlywheel :metatype mt_atomic>` |
| `MemorySizeInfo` not reflected | 2 failures |

A first attempt at the former deleted the registration outright and tripped
`-Werror=unused-local-typedefs`, so the suite ran against a STALE binary and
reported all-pass. Same trap as `.xo-backlog/xo-arena/issues/05`; the fix is to
make the falsification compile, here by guarding the call rather than removing
it.

`xo-build --sweep` ok both stages (71 attempted: 43 ok, 0 failed), umbrella
ctest 44/44, and the frame verified through the python binding.

## Not done

`type` still reads `_%sentinel%_` for every slot when a frame is produced from
python -- `.xo-backlog/xo-facet/issues/01`, untouched by any of this and now
the only field in a frame that misinforms.

No frame sequence number yet. The envelope is a printer now, which is where one
would go.
