# 17 -- introspect reads only "_members_"

Status: done 2026-10-11 -- umbrella `f4965945`
Type: task
Milestone: reflection-driven-json

The page reads these top-level keys, beside `_members_` (introspect.js,
`layout()` at :160-221):
- `snap.listen_port`, `snap.state`;
- `snap.endpoints`, and per endpoint `kind`, `stem`, `pattern`, `receiver`;
- `snap.sessions`, and per session `session_id`, `sender`, `sender.open`;
- `s.subscriptions`, and per subscription `sub_id`, `sink`, `stream`;
- `sink.sender._ref_`.

It never reads `refcount`, `has_receive`, `seq` or `sub.endpoint`.

Move it onto `_members_`:
- **labels** from members: `session_id_`, `stream_name_`, `sub_id_`,
  `uri_pattern_`, ..;
- **structure** from where each object is printed, in members:
  - endpoints under the url router's maps;
  - sessions under the session table;
  - subscriptions under the router's `subscription_v_`;
  - the sink under its subscription's `sink_`.

With first-encounter placement, the page must join by `_id_` / `_ref_`
wherever an object lands, rather than assume a fixed nesting.

This lets `issues/16` drop `endpoints[]`, `sessions[]`, `subscriptions[]`
and the view-model keys.

**Done when:** introspect works with the top-level view-model keys absent,
and the browser tests (`issues/14`) pass.

## Resolution (2026-10-11, umbrella `f4965945`)

- `layout()` reads `_members_` alone, as listed above: endpoints from
  `url_router_`'s `http_map_` / `stream_map_` (map key = stem), sessions
  from `session_table_`'s `session_map_` (key = session id), a session's
  subscriptions from `router_`'s `subscription_v_`; labels from
  `listen_port_`, `state_`, `kind_`, `uri_pattern_`, `open_`, `sub_id_`,
  `stream_name_`.  Each object is found by id (`objects_by_id`) wherever
  it landed.  Box ids, labels and edge kinds unchanged.
- **Beyond the ticket:** `layout()` works on a copy of the event, in which
  a boxed object found in place inside another's members becomes a ref
  (`hoist_to_refs`).  Without it, once 15 places sessions inside
  `session_map_`, a session would draw twice -- its own box, and opened in
  place in the map row (and as an extra nested box where it lands under a
  struct member, e.g. a sender under `router_`).
- **Proof of "keys absent":** the server cannot drop them yet (the bodies
  live in the view-model lists; `_unplaced_` carries refs only), so the
  new browser test `members_only.mjs` builds the post-15/16 shape in the
  page -- every view-model key stripped, each object moved to its first
  ref in member order -- and asserts `layout()` and the fully opened
  drawing (boxes, rows, ref edges) are identical to the live snapshot's.
  Checked to fail with the old `layout()` body, and with `hoist_to_refs`
  off.
- Test-side reads of view-model keys moved onto members too
  (`receiver.mjs`, `expand.mjs`, `sink.mjs`, `header.mjs`): they passed
  only because the server still writes the keys, and would have broken at
  16.  `sesstable_expand.mjs` expects `[map]` (left over from
  `xo-reflect/issues/05`).
- 21/21 browser tests pass.  Gotcha: the server serves a copy of
  `mount-origin/` made at build time -- rebuild `websock_ex_introspect`
  before `test/run.sh`, or it tests the stale page.
