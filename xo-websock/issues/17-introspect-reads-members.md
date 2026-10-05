# 17 -- introspect reads only "_members_"

Status: open
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
