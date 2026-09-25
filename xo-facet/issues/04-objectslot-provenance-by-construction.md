# 04 — prove an ObjectSlot's provenance, so a frame can report allocation size

Status: done 2026-09-25, umbrella `b6d4acca`
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

The proposal is to make both correct **by construction**: make a non-null
`ObjectSlot` something only a `DHandleStore` can create. See
[The design](#the-design-only-a-handlestore-can-create-an-objectslot) below.

**No `Evidence` token is carried.** Decided 2026-09-24: the chain of trust is
structural and legible from the types, so a witness would only restate it.

| step | who guarantees it | where |
|---|---|---|
| only a `DHandleStore` creates an `ObjectSlot` | the access restriction | `ObjectSlot.hpp:42-43`, private + friend |
| the pointer is in that store's storage arena | the store, per slot | `DHandleStore.hpp:230`, `storage_.contains` |
| that arena has headers, non-zero base align, and the agreed k | the store, once | `DHandleStore.hpp:98`,`:106`,`:112` |
| the arena was configured that way to begin with | `AllocFlywheel` | `xo-facet/src/facet/AllocFlywheel.cpp:63` |

This is the same move `.xo-backlog/xo-facet/issues/03` invariant 4 records for
`typerecd::_by_name`: *"The access restriction IS the lifetime contract."* Here
the access restriction is the provenance contract. Reading `ObjectSlot` tells
you a store made it; reading `DHandleStore` tells you what the store checked.

The `Evidence` / `EvidenceProvider` pattern
(`xo-subsys/include/xo/subsys/Evidence.hpp`) stays the fallback shape if the
store turns out not to be able to mint slots directly — it is already used by
`FacetAppcx` and, uncommitted, by `DHandleStore` itself for a DIFFERENT
proposition (that a `FacetAppcx` exists at all).

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

## The design: only a HandleStore can create an ObjectSlot

Decided 2026-09-24.

Make `ObjectSlot`'s value-carrying ctors private and befriend the `DHandleStore`
template — the forward-declared-friend shape already recorded in
`.xo-backlog/xo-facet/issues/02`. The default ctor stays public: a null slot
never reaches an arena (see the Measured section).

Construction then has to move out of the handle and into the store. That is the
whole production diff:

- `xo-facet/include/xo/facet/ObjectHandle.hpp:97` — `DObjectHandle::make_strong_ref`
  builds the slot and hands it over. It would instead hand over `obj<ATop>` and
  let `add_strong_ref` (`DHandleStore.hpp:230`) mint the slot.

**And the store can CHECK, not merely attest.** The obligation is already
written in prose one line above that function (`DHandleStore.hpp:228`):

> *Require: @p x refers to memory owned by @ref storage_*

Moving construction inside the store turns that comment into a runtime check
paid once per root creation, after which the type carries the result. A token
says "a validated store existed somewhere"; a check says "this pointer is in
*this* store's arena" — the same compile-time guarantee at the use site, a
materially better one at the creation site.

**Use `storage_.contains(p)`, not `DHandleStore::contains(p)`.** The latter
(`DHandleStore.hpp:189`) unions the root-set and free-list arenas, and those are
mapped by `DArenaVector::map` with `ArenaConfig::store_header_flag_` defaulting
to false. A pointer into them passes `contains` and then fails `alloc_info` —
precisely the failure this ticket exists to exclude.

### Why the arena is not passed down the print tree instead

Considered and rejected 2026-09-24. `obj2arena` is what makes a slot
**self-describing**, and that is the property worth keeping: `visit_object_slots`
hands slots out by reference to *any* consumer, so every walker of the root set
— not just `JsonPrinter_ObjectSlot` — would otherwise need the arena threaded to
it. Providing `DArena::obj2arena` is the reason that threading is unnecessary
(`.xo-backlog/xo-arena/issues/04`).

The mechanical obstacle is real too — printers dispatch by type key through
`provide_printer`, so `JsonPrinter_AllocFlywheel` has no parameter channel to
`JsonPrinter_ObjectSlot` — but it is the lesser reason.

### Cost: the ObjectSlotJson fixture

`root(DArena &, double)` at `xo-printjson/utest/ObjectSlotJson.test.cpp:118`
builds slots from a bare `DArena` with **no store at all**, and feeds three of
the file's five cases:

| line | case |
|---|---|
| 167 | `occupied-slot-reports-identity-and-offset` |
| 191 | `offset-is-resolved-from-the-pointer-alone` — two arenas, to prove the mask lands on the right one |
| 221 | `slot-without-an-agreed-alignment-reports-null-offset` |

Those need real stores, or need to move. This is the bulk of the work, and it is
arguably the point: the fixture constructs exactly the state the design makes
unrepresentable.

### It does not subsume the write-once gap

The two are orthogonal and both are needed. Store-only creation proves the
pointer lies in **some** validated store's storage arena. Whether masking then
*finds* that arena depends on `s_storage_base_align` still holding the value
that store was checked against at `DHandleStore.hpp:112` — and the printer reads
the global at PRINT time, long after the check.

## What landed, 2026-09-25 (umbrella `b6d4acca`)

| | |
|---|---|
| `ObjectSlot.hpp` | value-carrying ctors private, `DHandleStore` befriended through a forward declaration; default ctor stays public |
| `DHandleStore.hpp` | `add_strong_ref(const typename Handle::ATop *, void *)` mints the slot after `storage_.contains(data)`, throwing otherwise; `assign_storage_base_align` is write-once |
| `AllocFlywheel`, `DObjectHandle::make_strong_ref` | follow the new signature -- the whole of the production fallout |
| `JsonPrinter_ObjectSlot` | emits `size` from `arena->alloc_info(data).size()` |

```json
{"_name_": "ObjectSlot", "typeseq": 10, "type": "xo::scm::DFloat", "offset": 16, "size": 8}
```

verified through the python binding, not only the C++ suites.

### Two deviations from the plan above

**The byte-exact frame test needed no change.** It renders an EMPTY flywheel, so
no slot appears in it and the new key never reaches that expectation. `size` is
pinned instead in `occupied-slots-appear-in-the-frame`
(`xo-object2/utest/flywheel_frame.test.cpp`), spelled as
`"offset": 16, "size": 8` so it cannot accidentally match `RootSet`'s unrelated
`size` key one level up.

**xo-printjson's utest main gained a facet context.** The fixture needs a real
store; a store needs `FacetAppcxCreated`; and a second `Indentlog2Appcx` maps
another temp arena and installs another `PrettySinkFactory`, so each test file
owning one was not acceptable. `printjson_utest_main.cpp` now builds
`AppContext<S_indentlog2_tag, S_facet_tag>` as a function-local static, reached
by tests through a new `printjson_utest_appcx.hpp`. Unanticipated scope, and the
part a re-reader is most likely to trip over.

### Tests

`slot-without-an-agreed-alignment-reports-null-offset` is gone: it set the
global to 0, which now throws. Replaced by `storage-base-align-is-write-once`
and `a-store-refuses-a-foreign-pointer`. The printer's `align_z == 0` branch is
KEPT, with a comment recording that it is unreachable for a slot that exists --
`data` still arrives via `recover_native` from a TaggedPtr a caller assembled,
and what the guard prevents is a segfault rather than a wrong number.

Store-only creation is pinned by `static_assert(!std::is_constructible_v<...>)`;
`is_constructible` respects access, so the assert is the check.

**One done-when item is met only in mechanism.** The write-once test drives
`assign_storage_base_align` directly rather than constructing a second,
differing `FacetAppcx`. The guard is the same one either path reaches, but the
`FacetAppcx`-level case is not covered.

### Falsified, each compiling

| reverted | result |
|---|---|
| `storage_.contains` check | `a-store-refuses-a-foreign-pointer`: no exception thrown |
| write-once guard | `storage-base-align-is-write-once`: no exception thrown |
| `size` emission | 1 failure in printjson, 1 in object2 |
| private ctor | `static_assert` fires by name at `ObjectSlotJson.test.cpp:156` |

The last is a COMPILE failure rather than a red test, which is the stronger
form and not the stale-binary trap of issue 02: it names the assertion and the
message, so it cannot be mistaken for an unrelated build break.

### A prediction that was wrong

The first draft of the object2 expectation said `"size": 16`, reasoning that the
allocation includes its 8-byte `AllocHeader`. Observed 8:
`AllocInfo::size()` EXCLUDES the header, as
`xo-arena/include/xo/arena/AllocInfo.hpp:73` says. For `DFloat` that coincides
with `sizeof(DRepr)` -- fixed-size, no padding -- which is exactly why the
`DArray` / `DString` reasoning above is what justifies reading the header rather
than the type.

### Verification

```bash
cd .build && ctest                  # 45/45
xo-build --sweep                    # 71 attempted: 44 ok, 27 no tests, 0 failed
.build/xo-printjson/utest/utest.printjson "[ObjectSlot]"
.build/xo-object2/utest/utest.object2   "[flywheel]"
```

## Open questions

- Settled 2026-09-24, kept because it is the first thing a reader will ask:
  **`ObjectSlot` gains no member.** Evidence as a stored field would widen every
  slot; evidence as a ctor requirement is free at runtime; and neither is needed,
  because the chain of trust above is already legible from the types. See the
  table under the opening section.
- No `Milestone:` line: no existing milestone covers flywheel visualization.
  `xo-sdlc --milestones` as of 2026-09-24 lists ostream-containment,
  ppsink-migration, pyobject2, reflectable2.

## Files

- `xo-facet/include/xo/facet/handlestore/ObjectSlot.hpp:42-43` — value-carrying ctors, to go private
- `xo-facet/include/xo/facet/ObjectHandle.hpp:97` — the one production construction site, moves into the store
- `xo-facet/include/xo/facet/handlestore/DHandleStore.hpp:228-230` — `add_strong_ref`, gains the mint and the check
- `xo-facet/include/xo/facet/handlestore/DHandleStore.hpp:189` — `contains`, the one NOT to use
- `xo-facet/include/xo/facet/handlestore/DHandleStore.hpp:20`,`:98`,`:106`,`:112` — the setter and the three checks
- `xo-facet/src/facet/FacetAppcx.cpp:23` — the unguarded assignment
- `xo-facet/src/facet/DHandleStore.cpp:10` — `s_storage_base_align = 0` definition
- `xo-printjson/src/printjson/PrintJson.cpp:525` — `JsonPrinter_ObjectSlot`, gains `size`
- `xo-object2/utest/flywheel_frame.test.cpp` — byte-exact wire contract, must be updated deliberately
- `xo-printjson/utest/ObjectSlotJson.test.cpp:118` — the storeless fixture, and the three cases it feeds

## Done when

- a flywheel frame's non-null slots carry `size`, sourced from `AllocInfo::size()`
- a non-null `ObjectSlot` cannot be constructed outside a `DHandleStore`, and
  the store rejects a pointer its own `storage_` does not contain
- `DHandleStoreBase::assign_storage_base_align` rejects a conflicting
  reassignment, with a test that the second, differing `FacetAppcx` is refused
- `ObjectSlotJson.test.cpp`'s cases still cover what they cover today — in
  particular `offset-is-resolved-from-the-pointer-alone`, which is the only
  thing pinning that masking finds the right arena among several
- the byte-exact frame test is updated for the new key, deliberately
- falsified: remove the store-only restriction, the provenance check, or the
  write-once rule, and a test goes red — **and each falsification compiles**,
  per issue 02
- `xo-build --sweep` ok in both stages
