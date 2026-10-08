# 08 -- placement by ownership, not first encounter

Status: open
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
