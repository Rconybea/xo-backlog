# 16 -- retire the websock struct printers, one type at a time

Status: open (16a-16e done -- umbrella `860776d8`, `81b650de`, `80b20c45`, `e774091a`, `4752e002`; DynamicEndpoint, WebsocketSink left)
Type: task
Milestone: reflection-driven-json
Blocked by: none (`xo-printjson/issues/07` done; `xo-websock/issues/15` done 2026-10-11)

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
  introspect stops reading them (`issues/17` -- done 2026-10-11, umbrella
  `f4965945`; nothing in the page or its tests reads them now, and
  `members_only.mjs` checks the page without them).  The server's
  `"state"` key is the racy unlocked read noted in 15a: dropping it fixes
  that;
- the inline receiver (DynamicEndpoint) and the virtual
  `WebsocketSink::print_json`: decide whether the generic printer covers
  them, through self-tagging, or whether they stay recorded exceptions.

Placement follows ownership edges (`xo-printjson/issues/08`, revising the
milestone's first-encounter decision on 2026-10-08); where `pjson_` lands
is settled in `issues/15`.

**Done when:** no websock `JsonPrinter_*` remains except recorded
exceptions.

## Progress

**Member names (RC, 2026-10-11): rename as we go.**  `REFLECT_MEMBER(sr, x)`
registers `"x"` for member `x_` (`StructReflector.hpp:295`); the bespoke
printers re-add the underscore (`reflected_members(tp, "_")`), the generic
printer does not (`JsonPrintState.cpp:181`).  So retiring a printer drops
its type's trailing underscores on the wire.  Rejected: changing
`REFLECT_MEMBER` to register the declared name -- reflection is used
elsewhere, blast radius too high (29 files, 9 subsystems).  Mixed spellings
are accepted while 16 is in progress.

**refcount:** not a C++ member, so the generic printer cannot print it --
each retirement drops its type's `refcount` from Show JSON.  If refcounts
should stay visible, that needs deciding (e.g. a reflected computed
property) before 16 ends.

- **16a -- done, umbrella `860776d8`.**  `JsonPrinter_Webserver` retired: the
  server prints from reflection.  `_name_` `Webserver` -> `WebserverImpl`;
  view-model keys `refcount`, `listen_port`, `state` gone (and with `state`,
  the unguarded `state()` read of 15a); members `ws_config`, `pjson`,
  `url_router`, `session_table`, `listen_port`, `state`.  Golden: exactly
  those changes (checked by script).  Found: `box_subtitle()`
  (`introspect.js`) still read `d.obj.listen_port` / `.state` -- missed by
  `issues/17`, whose `members_only.mjs` compared rows but not box
  sub-titles; it now compares them too.  `noticker.mjs`'s "refcount fields
  remain" check moved to an endpoint (still bespoke).
- **16b -- done, umbrella `81b650de`.**  The three printers left as pure wrappers
  (`open_object` + `reflected_members(tp, "_")`, no keys) retired together:
  `JsonPrinter_UrlRouter`, `JsonPrinter_WsSessionTable`,
  `JsonPrinter_WebserverConfig`; `provide_url_router_json_printers` went
  with the first (nothing left to register).  Golden: exactly the renames
  (`http_map`, `stream_map`; `next_id`, `session_map`; `port`, `tls_flag`,
  `host_check_flag`, `use_retry_flag`, `mount_origin`) and WsSessionTable's
  `_name_` -> its short name `WsSessionTable<WebsocketSessionRecd>` (checked
  by script).  The live test's redaction list renamed with them -- `port`,
  `mount_origin` still `<redacted>` (else a run's port and a local path).
- **16c -- done, umbrella `80b20c45`.**  `JsonPrinter_WsSessionSender` and
  `JsonPrinter_WsSession` retired.  Reflection adds the sender's `target`
  (raw `WebserverImpl *`: borrowed, a ref -- its "not reflected" comment
  predated `xo-printjson/issues/08`) and the session's `sender` (`rp`,
  shared; the only placing edge -- the router's and sinks' `sender_` are
  hand-written refs -- so the sender stays under its session).
  `provide_webserver_json_printers` gone (empty); the session record's
  `session_id()` / `sender()` accessors too (only the printer used them --
  the build is the proof, the class is local to `Webserver.cpp`).  Golden:
  with ids relabelled (the sender now placed after the router), exactly the
  renames, `_name_`s, dropped keys (`session_id`, `sender`; `refcount`,
  `session_id`, `open`) and the sender's move into the session's `sender`
  member (checked by script).  `output_buf` stays redacted (renamed in the
  list).  A session's id now comes from its sender's `session_id`.
- **16d -- done, umbrella `e774091a`.**  `JsonPrinter_Subscription` retired;
  reflection adds `endpoint`, `sink` (shared `rp`s).  **Found:** a reflected
  `rp<WebsocketSink>` resolves to its most-derived type first
  (`WebsocketSinkImpl`, self-tagging), which had no printer -- so the
  generic printer took the sink, dropping its `sender_` / `pjson_` refs
  (the page's "astray" check reads the sink's `sender_`) and its keys.
  RC chose option A: `JsonPrinter_WebsocketSinkImpl` (`WebsocketSink.cpp`,
  `provide_websocket_sink_json_printers`) -- the same virtual
  `print_json`, so the sink prints as before; WebsocketSink's own
  retirement stays a separate decision.  Golden: exactly the dropped keys,
  renames, and the sink's move into `sink` (ids unchanged; checked by
  script).  Behaviour change outside server prints: a subscription printed
  ALONE now places its endpoint in full (shared, first encounter) where
  the hand-written ref did not; `WsSessionRouter.test.cpp` checks the new
  shape.
- **16e -- done, umbrella `4752e002`.**  `JsonPrinter_WsSessionRouter` retired
  (RC: option A).  `url_router_` changed from `UrlRouter const &` to
  `UrlRouter const *` -- a reference member has no member pointer, so
  `REFLECT_MEMBER` cannot name it; the ctor still takes a reference.
  Reflection adds `url_router` (raw pointer: borrowed), `sender`,
  `pjson` (`.borrowed()` -- the router prints before the session's sender,
  so shared would have placed the sender inside the router).
  `provide_router_json_printers` gone (empty).  **Found:** the borrowed
  `rp<WsSender>` refs landed in `_unplaced_` -- the sender, placed as
  `WsSessionSenderImpl` at the same address, was not known to derive from
  `WsSender`; fixed by reflecting `WsSender` (no members) and
  `adopt_parent<WsSender>()` in the sender's reflection.  Golden: with ids
  relabelled, exactly the renames, reorder (unguarded first:
  `readjson`, `url_router`, `sender`, `pjson`, then `subscription_v`) and
  `url_router`'s type `const UrlRouter&` -> `const UrlRouter*`.
  `rows.mjs` (`issues/18`) now checks the const-pointer member.

