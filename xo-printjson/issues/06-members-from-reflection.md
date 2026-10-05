# 06 -- write "_members_" from a type's reflection

Status: done 2026-10-05 -- umbrella `0767d431`..`351d5609`
Type: feature
Milestone: reflection-driven-json
Blocked by: `.xo-backlog/xo-websock/issues/14`

Add to `JsonMembers` a way to write one `_members_` entry per reflected member
of a struct, from its `StructReflector` description: name, declared type,
metatype, and value via the member accessor. This is the same entry a
hand-written `.member("x_", obj->x_)` writes today.

Then reflect the plain data members of each websock type. Today all but
`WebserverConfig` and `WebsocketSinkImpl` are reflected with no members
(`Webserver.cpp:2279-2290`, `WsSessionRouter.cpp:522-536`,
`DynamicEndpoint.cpp:143-149`, `UrlRouter.cpp:251-257`). Replace their
printers' runs of `.member(...)` with the reflected ones. `member_as` and
`member_ref` stay where they are needed.

**Done when:** the golden snapshot (`xo-websock/issues/14`) is unchanged,
and `JsonPrinter_WebserverConfig` has no hand-written members.

## Done, 2026-10-05 -- umbrella `0767d431`..`351d5609`

**`JsonMembers::reflected_members(obj, name_suffix = {})`** (`0767d431`;
the suffix arrived with `44f47447`):
- It writes one entry per member reflected for `obj`'s type, in the order
  xo-reflect holds them. It walks the `StructMember`s by index, O(1) per
  member: RC asked for this in place of a per-name lookup, which would be
  O(n) per member.
- Each entry is what `member()` writes: the declared type and metatype from
  `get_member_td()`, the value from `get_member_tp()` with identity. A
  member that cannot print gets the same "type not reflected" `_error_`
  entry.
- Printability is checked on the value: through a pointer to its target,
  through a vector to its first element.

**Names.** Reflection keeps the house convention: `REFLECT_MEMBER(sr, port)`
reflects `port_` as `"port"`. The json printer appends the underscore
back, with `name_suffix = "_"`, so `_members_` keeps naming C++ members
(RC). Stripped names, or a runtime option, can come later.

**Types moved**, one commit each. Each reflects its plain members only,
with the private ones through an in-class static `reflect_self`:

| Type | Commit | Reflected | Golden diff |
|---|---|---|---|
| WebserverConfig | `44f47447` | all 5 (no hand-written members left) | none |
| WebsocketSinkImpl | `07ea7fd7` | stream_name_, sub_id_, n_in_ev_ (sender_, pjson_ removed from reflection: refs) | reorder |
| WsSessionSender | `e56df166` | session_id_ | reorder |
| Subscription | `473bd09e` | sub_id_, stream_name_ | none |
| DynamicEndpoint | `eea008ca` | uri_pattern_, var_v_ | reorder |
| WsSession (WebsocketSessionRecd) | `fe732c63` | router_ | reorder + id renumbering |
| Webserver (WebserverImpl) | `e8d71991`, `351d5609` | ws_config_, pjson_, url_router_, session_table_ | reorder |

- **Reflected members come first** in `_members_`, then the hand-written
  `member_as` / `member_ref` entries. Where a printer interleaved them, the
  order changed.
- **WsSession's ids were renumbered.** Ids follow first mention, and
  `router_` now prints before each session's `OutputBuffer` pointee. A
  script checked the change:
  - ignoring ids and member order, the two snapshots are identical;
  - one consistent, one-to-one map takes each old id to its new one, for
    `_id_` and `_ref_` alike.
- **Left out of reflection on purpose** (each type's comment says which and
  why):
  - members printed elsewhere, written as refs (placement);
  - atomics, enums, deque, `unique_ptr`, `std::function`, regex,
    `CallbackId` (`xo-reflect/issues/04`);
  - lock-guarded copies (`xo-websock/issues/15`);
  - references (`url_router_` in WsSessionRouter: a reference member has
    no member pointer).
- **WsSessionRouter, UrlRouter and WsSessionTable** have no plain members,
  so they are unchanged here.
- **Tests that read `_members_` by position now look members up by name**
  (`value_of` / `entry_of` helpers) where the order moved:
  `Webserver.test.cpp`, `WebserverLive.test.cpp`, and the browser tests
  sink, sender_expand, session_expand and expand.

Checked at each step: the golden snapshot (`xo-websock/issues/14`), ctest
49 / 49, and the 20 browser tests.
