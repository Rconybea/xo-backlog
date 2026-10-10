# 15 -- declare websock ownership and guards; printers stop locking

Status: open
Type: task
Milestone: reflection-driven-json
Blocked by: `.xo-backlog/xo-printjson/issues/09`

(Filename kept from the superseded plan below, so references still resolve.)

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
