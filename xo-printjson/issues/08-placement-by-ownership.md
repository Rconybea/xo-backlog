# 08 -- placement by ownership, not first encounter

Status: done 2026-10-10 -- umbrella `ca5a871f`, `59d483b5`
Type: feature
Milestone: reflection-driven-json
Blocked by: `.xo-backlog/xo-reflect/issues/07`

Today the generic printer places an object in full where the traversal
first reaches it (`issues/02`, milestone decision 2026-10-05).  Replace that
with the edge kinds from `xo-reflect/issues/07` (RC, 2026-10-08):

- **owning** edge: place in full.  Reaching an already-placed object through
  an owning edge means two owners -- a bug: assert.
- **shared** edge (`rp<T>`): place at first appearance, a `_ref_` afterwards.
- **borrowed** edge (`T*`): always a `_ref_`; never dereference -- a
  dangling borrowed pointer prints harmlessly.

Bespoke printers' explicit `member_ref` / `member_refs` / `member_ref_map`
(`include/xo/printjson/JsonMembers.hpp:109-137`) keep working.

## Two wrinkles

- **Ref identity without dereferencing.**  The identity map matches by
  address and type (`JsonPrintState::is_printed(TaggedPtr)`,
  `include/xo/printjson/JsonPrintState.hpp:111`).  A borrowed edge knows only
  the static pointee type -- `most_derived_self_tp` would read the pointee.
  So a ref matches a placed entry at the same address whose type is the
  static type or derived from it.  Wrong for a non-primary base under
  multiple inheritance (different address); acceptable to start, assert
  where detectable.
- **Unplaced refs.**  An object reachable only through borrowed edges is
  never placed.  RC's earlier objection to owner placement (2026-10-05) was
  exactly this.  Resolution: the ref stays a ref, and the output gains a
  trailer listing them, without reading them:

  ```json
  "_unplaced_": [{"_id_": 7, "_type_": "...", "_address_": "0x..."}]
  ```

  A missing ownership annotation then shows up in output (and in a golden
  test asserting the trailer is empty) instead of as silent misplacement.

Placing orphans at the end instead was considered and rejected: it reads
objects outside any owner's lock.

**Done when:** placement follows edge kind, `_unplaced_` lists exactly the
never-placed ref targets, and the xo-printjson tests cover all three kinds
plus a ref to a derived object through a base pointer.

## As built (umbrella `ca5a871f`, `59d483b5`)

- **Ref identity -- revised (RC, 2026-10-09).**  The "static type or
  derived" match above needed a base-class relation xo-reflect lacked.
  `ca5a871f` adds it: `StructTdx` records parents,
  `StructReflector::adopt_parent<P>()` replaces `adopt_ancestors` (and
  static_asserts the base), `TypeDescr::is_derived_from()` walks them.  The
  identity map stays keyed by address; whether a borrowed ref's target is
  placed is decided when the top-level object closes: placed iff the placed
  type `is_derived_from` the ref's pointee type.  So a borrowed pointer to a
  struct's first member is reported, and a base declared without
  `adopt_parent` is a visible false positive (`WebserverImpl` now adopts
  `Webserver` for exactly this).
- `JsonPrintState::print_pointee(ptr, edge)`; borrowed reads only the
  pointer slot, via new `PointerTdx::pointee_address()` (xo-reflect; also
  `FopTdx`, xo-reflectable2).  `JsonMembers::reflected_members` passes
  `StructMember::ownership()`, so member overrides take effect.
- `FopTdx::child_edge_ownership()` is `shared`: fops live in a collected
  heap, share and cycle.
- **Trailer entries carry `_ref_`, not `_id_` (RC, 2026-10-09):** an entry
  names its object without defining it.  Invariant, checked by introspect's
  `refs_resolve.mjs`: every ref resolves to an `_id_` or is listed in
  `_unplaced_`; no `_id_` twice.  Written by the top-level object's
  `close()`, only when non-empty; `_address_` decimal (golden test redacts
  it).
- **A top-level pointer places its target.**  Entry points
  (`PrintJson::print_tp`, `validate_tp`) call private
  `JsonPrintState::print_root()`: the caller vouches for a pointer it hands
  in.  Explicit, after a depth-inferred version misfired on a bare
  `JsonPrintState`.
- Raw-pointer fixtures that meant "print the pointee" now say so:
  `.shared()` in the printjson cycle/diamond tests, `.owning()` on
  `HoldsServer::server_` and `IntrospectSnapshot::server_`.
- Websock golden: `_unplaced_` lists the two `OutputBuffer`s behind
  `output_buf_` (borrowed via `member_as`) -- see `xo-websock/issues/15`.
- Tests: `xo-printjson/utest/PrintJsonOwnership.test.cpp`,
  `xo-reflect/utest/AdoptParent.test.cpp`.
