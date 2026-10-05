# 16 -- retire the websock struct printers, one type at a time

Status: open
Type: task
Milestone: reflection-driven-json
Blocked by: `.xo-backlog/xo-printjson/issues/07`, `.xo-backlog/xo-websock/issues/15`

Move each websock type onto the generic printer and delete its
`JsonPrinter_*`, simplest first:
1. WebserverConfig
2. WsSessionTable
3. UrlRouter
4. WsSessionRouter
5. Subscription
6. WsSessionSender
7. DynamicEndpoint
8. WsSession
9. Webserver

Each move shows its intended diff in the golden snapshot
(`issues/14`), and the browser tests stay green.

What blocks some of them:
- maps (UrlRouter, WsSessionTable): `xo-reflect/issues/05`, or keep a
  custom printer meanwhile;
- atomics, enums, deque, `unique_ptr`: `xo-reflect/issues/04`;
- view-model top-level keys: the printer can drop them only after
  introspect stops reading them (`issues/17`);
- the inline receiver (DynamicEndpoint) and the virtual
  `WebsocketSink::print_json`: decide whether the generic printer covers
  them, through self-tagging, or whether they stay recorded exceptions.

Placement becomes first encounter (milestone decision). `pjson_` will land
inside the first sink again; accept that.

**Done when:** no websock `JsonPrinter_*` remains except recorded
exceptions.
