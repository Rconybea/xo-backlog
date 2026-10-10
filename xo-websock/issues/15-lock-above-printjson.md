# 15 -- declare websock ownership and guards; printers stop locking

Status: open (15a, 15b, 15c done -- umbrella `b83bc579`, `617b5f68`, `e4683ed8`; rest blocked)
Type: task
Milestone: reflection-driven-json
Blocked by: `.xo-backlog/xo-reflect/issues/05`, `.xo-backlog/xo-websock/issues/17`

(Filename kept from the superseded plan below, so references still resolve.)

## Slices (2026-10-10)

Not doable in one pass: most of it waits on other tickets.

| locking printer | lock | what reflection needs | blocked by |
|---|---|---|---|
| WsSessionTable | table `mutex_` | `session_map_`: `unordered_map<id, unique_ptr<Recd>>` | `xo-reflect/issues/05` (maps) |
| UrlRouter | router `mutex_` | `http_map_`, `stream_map_`: maps | `xo-reflect/issues/05` |
| ~~WsSession~~ | session `mutex_` | `outbound_q_`: `std::deque` | done: 15c (`e4683ed8`) |
| WsSessionRouter | router `mutex_` | `subscription_v_`: `vector<unique_ptr<Subscription>>` (reflectable now) | `xo-websock/issues/17` |

**The view-model lists place first.**  The server's `endpoints[]` /
`sessions[]`, a session's `subscriptions[]` and a subscription's `"sink"`
print objects in full, each under its own lock-holding visitor, BEFORE the
reflected members.  Once `subscription_v_` / `session_map_` are owning
reflected edges, they reach objects placed already -- the two-owners
assert (`xo-printjson/issues/08`).  The lists can become refs or go only
once introspect reads `_members_` alone (`issues/17`): so 17 precedes the
rest of 15, not follows it.

**Caller rule checked (2026-10-10):** introspect prints inside
`IntrospectReceiver::receive()`, which runs with no websock lock held --
`perform_ws_cmd` uses `find_owner_thread` (table lock released before it
returns, `Webserver.cpp:1620`), and the router dispatches `receive` without
its lock (`WsSessionRouter.cpp:266`).

- **15a -- done, umbrella `b83bc579`.**  `WebserverImpl::state_`
  `.guarded_by(mutex_)`: fixed a race -- `reflected_members` read `state_`
  unlocked while start/stop write it under `mutex_`.  `state_` now prints
  last among the server's members (golden: reorder only, checked by
  script).  Still racy: the view-model `"state"` key reads it via the
  unlocked `state()` accessor; it goes with `issues/17`.
- **15b -- done, umbrella `617b5f68`.**  Session record: `output_buf_`
  reflected `.owning().guarded_by(&mutex_)`, `last_msg_seq_`
  `.guarded_by(&mutex_)`; the printer's `member_as<OutputBuffer *>` and its
  locked copy dropped.  `outbound_q_` stays a size summary, copied under the
  same mutex, released before `reflected_members` takes it (sequential, not
  nested), until the deque is reflected.  Golden `_unplaced_` empty, and
  the golden test asserts it absent; `session_expand.mjs` shows
  `output_buf_: OutputBuffer` again, plus `last_msg_seq_`.
- **15c -- done, umbrella `e4683ed8`.**  `outbound_q_` reflected
  `.guarded_by(&mutex_)` (deque reflection, `xo-reflect/issues/04`): prints
  its queued messages.  The session printer takes no lock.
- **Rest** (blocked as tabled above), in order: `xo-reflect/issues/05`,
  then `issues/17`, then the remaining types here (WsSessionTable,
  UrlRouter, WsSessionRouter), then `issues/16`.

Today these printers take their own mutex while reading:
- WsSessionRouter: `subscription_v_` (`WsSessionRouter.cpp:470`);
- UrlRouter: its maps (`UrlRouter.cpp:280`);
- WsSessionTable: `next_id_`, `session_map_` (`Webserver.cpp:1252`);
- WsSession: `output_buf_`, `outbound_q_` (`Webserver.cpp:1144`).

```bash
grep -n "lock_guard.*(r->mutex_)\|lock_guard.*(t->mutex_)\|recd->mutex_" \
    xo-websock/src/websock/*.cpp
```

## Plan (RC, 2026-10-08)

Annotate the websock types with ownership (`xo-reflect/issues/07`) and guards
(`xo-reflect/issues/08`), and let the guard-aware generic printer
(`xo-printjson/issues/09`) take the locks:

- `.guarded_by` on every member a websock mutex guards -- the list is in
  `xo-reflect/issues/08`;
- ownership overrides where a default misplaces.  Candidate:
  `WsSessionRouter::pjson_` as `.borrowed()`, so the server's `pjson_` is its
  home -- decide when the golden diff shows placement;
- the caller of the server print holds no websock guard (find the introspect
  publish path and check);
- the golden test (`issues/14`) asserts `_unplaced_` is empty.

- `output_buf_`: a raw `OutputBuffer *`, borrowed by default, so since
  `xo-printjson/issues/08` (umbrella `59d483b5`) it prints a ref and its two
  `OutputBuffer`s appear in the golden snapshot's `_unplaced_`;
  introspect's `session_expand.mjs` expects the ref.  Decide its home (an
  `.owning()` override once it is reflected, guarded by the session mutex)
  so the trailer goes empty.

**In one change per type:** since `xo-printjson/issues/09` (umbrella
`3bbda0b6`), `reflected_members` takes a struct's declared guards -- so a
bespoke printer that locks its own mutex and then calls
`reflected_members` would deadlock once that type declares `.guarded_by`.
Add the declaration and remove the printer's lock together.

Then the printers above need not lock, which unblocks retiring them
(`issues/16`).  The deque (`outbound_q_`, `xo-reflect/issues/04`) can then
print through reflection under its guard.

## A consistency hazard this fixes

The server prints `sessions[]` under one acquisition of the table lock
(`WebserverImpl::visit_sessions`, `Webserver.cpp:1051`) and the table's
`session_map_` refs under another (`Webserver.cpp:1252`).  A session opening
between the two leaves a `_ref_` with no target; one closing between them
vanishes from the map.  Once `session_map_`'s `unique_ptr`s are owning
edges, each session prints inside the table, under one acquisition.

Not a dangling read.  An earlier reading (2026-10-08, in conversation) said
the table printer reads sessions after unlocking.  It does not:
`member_ref_map` writes refs by address and never dereferences
(`xo-printjson/include/xo/printjson/JsonMembers.hpp:133`).

## Superseded plan (2026-10-05)

"Take locks above printjson, not inside printers": the caller holds the
locks for the whole print, and no printer locks.  RC replaced it on
2026-10-08: callers must hold *no* guard (non-recursive mutexes), and
locking moves into reflection declarations.  The old plan was plausible
while printjson was meant to stay lock-free; it would also have made a
caller know every mutex in the traversed structure.

**Done when:** websock's guarded members carry `.guarded_by`, no websock
printer takes a lock, `_unplaced_` is empty in the golden snapshot, and the
browser tests stay green.
