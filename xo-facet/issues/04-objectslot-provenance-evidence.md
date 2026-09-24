# 04 — prove an ObjectSlot's provenance, so a frame can report allocation size

Status: open
Type: feature

A flywheel frame's slots carry `typeseq`, `type` and `offset`. They should also
carry **size**, so a consumer can draw a cell proportional to what the object
actually occupies.

The number wanted is `AllocInfo::size()` —
`xo-arena/include/xo/arena/AllocInfo.hpp:73`, "allocation size (including
allocator-supplied padding, excluding alloc header)". Reaching it from a slot
means two calls that are today correct by convention:

```cpp
DArena * arena = DArena::obj2arena(data, align_z);   // recover the arena
arena->alloc_info(data);                             // read its header
```

The proposal is to make both correct **by construction**: give `ObjectSlot` a
compile-checked witness that it refers to AllocFlywheel-owned storage, using the
`Evidence` / `EvidenceProvider` pattern already in the tree
(`xo-subsys/include/xo/subsys/Evidence.hpp`).

## Why `sizeof(DRepr)` is not the answer

It is available — `DObjectHandle<AFacet, DRepr>::make_strong_ref`
(`xo-facet/include/xo/facet/ObjectHandle.hpp:89`) has `DRepr` statically, and
`ATop::_drop` already shows the vtable route for per-representation constants
(`xo-facet/include/xo/facet/alloc/IAllocator_Xfer.hpp:43`).

But it is the representation size, not the allocation size. `DArray` is
documented at `xo-object2/include/xo/object2/DArray.hpp:37` as having "max
capacity fixed at construction time, but not part of type"; `DString` is the
same shape. For those, `sizeof(DRepr)` is a fixed header that says nothing about
the payload, and a proportional cell drawn from it would be wrong by an
unbounded factor. The alloc header is the only thing that knows.

## The two obligations, and a misreading worth keeping

`alloc_info()` is sound iff the arena has `store_header_flag_`. That is a
per-arena fact.

`obj2arena()` looked harder. `xo-arena/include/xo/arena/DArena.hpp:110-117`
states its requirement as "all arenas to which `x` might belong must be using
the same value for `base_align_z_`" and adds "the requirement isn't verifiable
at runtime" — which reads as a property of every arena in the process, and so
as something no single arena could witness.

**That reading is wrong, and it is worth recording because it is the natural
one.** `.xo-backlog/xo-arena/issues/04` draws the distinction the sentence
omits: *"Alignment alone recovers the base of a pointer ALREADY KNOWN to belong
to this arena."* The unverifiable case is **classifying** an anonymous pointer.
A slot whose provenance is proven needs only **recovery**, and recovery holds
per-arena, because `DArena::map` already validates `size_ <= base_align_z_` — so
masking a pointer inside arena A yields A's base whatever else is mapped.

So the proof obligation collapses to one claim: *this slot points into
flywheel-owned storage*.

## The anchor already exists

`DHandleStore`'s ctor validates all three conditions in one place
(`xo-facet/include/xo/facet/handlestore/DHandleStore.hpp`):

| line | check |
|---|---|
| 98 | `store_header_flag_` set — else throw |
| 106 | `base_align_z_` non-zero — else throw |
| 112 | `base_align_z_ == DHandleStoreBase::s_storage_base_align` — else throw |

Participation is enforced at the door, not assumed. That ctor is the natural
`EvidenceProvider`.

Work in this direction is already in the working tree, uncommitted as of
2026-09-24 at `6ff7c722`: `DHandleStore` now takes a `FacetAppcxCreated` by
value and stores it (`DHandleStore.hpp:90`, `:130`, `:323`), passed by
`AllocFlywheel`'s ctor from `facet_appcx.creation_evidence()`
(`xo-facet/src/facet/AllocFlywheel.cpp:23`).

```bash
git diff xo-facet/include/xo/facet/handlestore/DHandleStore.hpp
```

## The gap: `s_storage_base_align` is not write-once

The check at `:112` compares against a mutable global with a public, unguarded
setter:

```cpp
// DHandleStore.hpp:20
static void assign_storage_base_align(std::size_t z) { s_storage_base_align = z; }
```

A plain assignment cannot reject anything. `FacetAppcx`'s ctor calls it
unconditionally (`xo-facet/src/facet/FacetAppcx.cpp:23`), and `FacetAppcx` has
no singleton enforcement — its ctor is public and it holds no static state
(`xo-facet/include/xo/facet/cx/FacetAppcx.hpp:33`).

So "runtime would have aborted on an attempt to construct a conflicting
AllocFlywheel" holds only for the flywheel side. The sequence that defeats it:

1. `FacetAppcx` #1, align 1MB → `s_storage_base_align = 1MB`
2. flywheel A constructed; `:112` passes
3. `FacetAppcx` #2, align 2MB → global silently reassigned. **No check, nothing
   throws.** This is the only unguarded step.
4. flywheel B constructed; `:112` passes against the *new* value
5. slots from A and B coexist; `JsonPrinter_ObjectSlot` reads
   `storage_base_align()` = 2MB and masks A's pointers with the wrong k

Step 3 is not hypothetical plumbing — a test already drives the setter directly,
including to 0:

```bash
grep -n assign_storage_base_align xo-printjson/utest/ObjectSlotJson.test.cpp
#  99:  DHandleStoreBase::assign_storage_base_align(c_align);
# 237:  DHandleStoreBase::assign_storage_base_align(0);
```

**Making that assignment write-once — reject a conflicting value rather than
overwrite — is what turns the argument into a proof.** The global then becomes
constant after first assignment, every participating arena is checked against it
at construction, and a slot carrying evidence genuinely establishes both calls.

It also collapses the printer's `align_z == 0` branch
(`xo-printjson/src/printjson/PrintJson.cpp:566`) into something reachable only
without evidence.

Note the test at `:237` sets 0 deliberately, to exercise that branch, and
restores the previous value at `:241`. A write-once rule has to decide what it
does about a test that needs to put the global back — an explicit reset entry
point, or the branch and its test go away together.

## Measured 2026-09-24

**`ObjectSlot`'s default ctor can be removed without touching production code.**

```bash
sed -i 's/ObjectSlot() = default;/ObjectSlot() = delete;/' \
    xo-facet/include/xo/facet/handlestore/ObjectSlot.hpp
cmake --build .build -j8 2>&1 | grep -E 'error:'
```

One error in the whole tree, and it is a test:

```
xo-printjson/utest/ObjectSlotJson.test.cpp:158:24:
  error: use of deleted function 'xo::facet::ObjectSlot::ObjectSlot()'
```

— `ObjectSlot empty;` in `empty-slot-renders-as-null`.

**A predicted second failure did not occur, and the wrong reason is instructive.**
The expectation was that `DArenaVector<T>::resize` would force it: its body
contains `new (addr) T()` at `xo-arena/include/xo/arena/DArenaVector.hpp:334`,
and `DHandleStore::clear` calls `strong_refs_.clear()`
(`DHandleStore.hpp:241-243`), which routes to `resize(0)`. Instantiating a
function body requires `T()` to be valid even on a branch that never runs.

It does not fire because `DHandleStore::clear` is a member of a class template
and is **never called** — so `DArenaVector<ObjectSlot>::clear`, and with it
`resize`, are never instantiated. Plausible, cheap to check, and false.

Consequence: the invariant to enforce is over **non-null** slots only. A cleared
slot is produced by `refs[ix].reset()` (`DHandleStore.hpp:289`), and the printer
returns `null` before touching an arena. The surface to guard is the two
value-carrying ctors at
`xo-facet/include/xo/facet/handlestore/ObjectSlot.hpp:42-43` — the second,
`explicit ObjectSlot(const obj<ATop> &)`, being the one that would otherwise
launder an arbitrary object into a slot.

**Probe hygiene.** A first attempt at the above inserted `#error` at line 15 of
`ObjectSlot.hpp` to confirm the build reads the source header, saw no error, and
nearly concluded the build was stale. Line 15 is inside the class's doxygen
comment block. The probe has to land after `#pragma once` (line 7), where it
does fire twice. Same family as the stale-binary trap in
`.xo-backlog/xo-facet/issues/02`: a falsification that cannot fail is not a
falsification.

## Open questions

- **Stored or only required?** Evidence in `ObjectSlot` as a member widens every
  slot; evidence as a ctor *parameter* is free at runtime and is what "every
  non-null slot was built from proven storage" actually needs. The latter proves
  it at each construction site rather than carrying it.
- **How does the arena reach the printer?** Printers are dispatched by type key
  through `provide_printer`, so `JsonPrinter_AllocFlywheel` has no parameter
  channel to `JsonPrinter_ObjectSlot`. If one existed, the flywheel could hand
  down `storage()` (`xo-facet/include/xo/facet/AllocFlywheel.hpp:61`) and
  `obj2arena` would leave the path entirely — a stronger outcome than proving
  it sound. Worth deciding before building the witness, since it may make part
  of it unnecessary.
- No `Milestone:` line: no existing milestone covers flywheel visualization.
  `xo-sdlc --milestones` as of 2026-09-24 lists ostream-containment,
  ppsink-migration, pyobject2, reflectable2.

## Files

- `xo-facet/include/xo/facet/handlestore/ObjectSlot.hpp:41-43` — ctors to guard
- `xo-facet/include/xo/facet/handlestore/DHandleStore.hpp:20`,`:98`,`:106`,`:112` — the setter and the three checks
- `xo-facet/src/facet/FacetAppcx.cpp:23` — the unguarded assignment
- `xo-facet/src/facet/DHandleStore.cpp:10` — `s_storage_base_align = 0` definition
- `xo-printjson/src/printjson/PrintJson.cpp:525` — `JsonPrinter_ObjectSlot`, gains `size`
- `xo-object2/utest/flywheel_frame.test.cpp` — byte-exact wire contract, must be updated deliberately
- `xo-printjson/utest/ObjectSlotJson.test.cpp:99`,`:158`,`:237` — the three sites this ticket disturbs

## Done when

- a flywheel frame's non-null slots carry `size`, sourced from `AllocInfo::size()`
- `DHandleStoreBase::assign_storage_base_align` rejects a conflicting
  reassignment, with a test that the second, differing `FacetAppcx` is refused
- a non-null `ObjectSlot` cannot be constructed without a witness that its
  storage came from a validated `DHandleStore`
- the byte-exact frame test is updated for the new key, deliberately
- falsified: remove the witness requirement, or the write-once rule, and a test
  goes red — **and the falsification compiles**, per issue 02
- `xo-build --sweep` ok in both stages
