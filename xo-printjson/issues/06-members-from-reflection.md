# 06 -- write "_members_" from a type's reflection

Status: open
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
