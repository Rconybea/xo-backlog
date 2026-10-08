# 08 -- guard declarations: which mutex guards which members

Status: done 2026-10-08 -- umbrella `ea6eacf4`
Type: feature
Milestone: reflection-driven-json
Blocked by: `.xo-backlog/xo-reflect/issues/07`

Let a reflected struct say which of its members a mutex guards, so a
generic traversal can take that mutex while it reads them
(`xo-printjson/issues/09`).  RC (2026-10-08): printjson takes locks, but only
those reflection declares; it never names a mutex.  Introspection first.

## Design (RC, 2026-10-08)

- **Lockables are a capability**, like `StdAtomicTdx`
  (`include/xo/reflect/atomic/StdAtomicTdx.hpp:28`): `LockableTdx`,
  `mt_atomic` with no children, reached by `lockable_info()` /
  `is_lockable()`, installed by `EstablishTdx<std::mutex>` and
  `EstablishTdx<std::shared_mutex>`.  Phrased for a reader:
  `read_lock(void*)`, `read_unlock(void*)`, `try_read_lock(void*)` --
  `lock`/`unlock`/`try_lock` on a `std::mutex`, `lock_shared` & co on a
  `std::shared_mutex`.
- **Declared per member**, on the builder from `xo-reflect/issues/07`,
  mirroring clang's `GUARDED_BY`:

  ```cpp
  REFLECT_MEMBER(sr, outbound_q).guarded_by(&Recd::mutex_);
  ```

  The guard need not be a reflected member (mutexes stay off the introspect
  page).  `guarded_by` takes `MutexT OwnerT::*` with `StructT` derived from
  `OwnerT`, as `GeneralStructMemberAccessor` does; it static_asserts a
  lockable type.
- **Interned per struct.**  `StructTdx` (`include/xo/reflect/struct/StructTdx.hpp:19`)
  gains a guard table -- accessors to lockables, one per distinct member
  pointer -- and each `StructMember` a guard index, or none.
- **Sibling guards only, for now.**  A member guarded by a lock in some other
  object is out of scope until a case appears.
- **A helper for traversals:** for a struct object, visit its unguarded
  members, then for each guard in declaration order: lock, visit that
  guard's members, unlock.  Sibling guards therefore never nest; nesting
  happens only by descending owning/shared edges, so acquisition order is
  ownership depth.  Two modes: blocking, and try (a busy guard's members are
  reported as unavailable).

## Scope check: every websock mutex guards siblings

- `WsSessionTable::mutex_` -- `next_id_`, `session_map_`;
- `UrlRouter::mutex_`, `WsSessionRouter::mutex_` -- their own maps / vectors;
- `WebsocketSessionRecd::mutex_` -- `output_buf_`, `last_msg_seq_`,
  `outbound_q_` (`xo-websock/src/websock/Webserver.cpp:600-604`);
- `WebserverImpl::cx_mutex_` -- `lws_cx_`; `mutex_` -- `state_`;
  `removed_mutex_` -- `removed_endpoint_v_`.

```bash
grep -rn "std::mutex .*_;" xo-websock/include xo-websock/src
```

Out of scope: thread-confined state with no mutex (a later `.confined()`);
recursive mutexes.

**Done when:** a struct with two guards and some unguarded members reports
each member's guard, the helper visits them grouped with each guard held
for exactly its group, and try mode reports a group whose guard another
thread holds.

## As built (umbrella `ea6eacf4`)

- `LockableTdx` (`include/xo/reflect/lockable/LockableTdx.hpp`), installed
  for `std::mutex` (exclusive) and `std::shared_mutex` (shared) in
  `Reflect.hpp`; `lockable_info()` / `is_lockable()` on `TypeDescr` and
  `TypeDescrExtra`.
- `StructTdx` holds the guard table (accessors) plus member indices grouped
  by guard, computed once at construction; `n_guard()` / `guard_tp(g, obj)`
  on `TypeDescr` and `TypeDescrExtra`.  `StructMember::guard_ix()` is
  `std::optional<uint32_t>`.
- Interning: `AbstractStructMemberAccessor::same_member()` -- member-pointer
  equality, no `offsetof`.
- `StructMemberDecl` now holds the reflector and the member's index (no
  longer a raw `StructMember *`), so it stays valid while the reflector does.
  `.guarded_by()` static_asserts the guard belongs to `StructT` or a base,
  and asserts a reflected lockable and at most one guard per member.
- `adopt_ancestors` appends the ancestor's guards and shifts adopted
  indices.  Known limit: a mutex both the derived class and an ancestor
  name becomes two guards (taken in turn, never nested).
- Helper: `visit_members_guarded(td, obj, GuardMode, fn(ix, readable))` in
  `include/xo/reflect/struct/GuardedVisit.hpp` -- unguarded first, then
  each guard's group with it held; RAII release; `try_lock` mode reports a
  busy group unreadable.
- Tests: `xo-reflect/utest/Guard.test.cpp` (cross-thread probes).
