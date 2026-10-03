# 05 — `_type_` -> `_canonical_type_` + `_short_type_`

Status: done 2026-10-03 -- umbrella `3ee09e14`
Type: feature

RC: rename the json key `_type_` to `_canonical_type_` and add `_short_type_`,
both forwarding the corresponding `TypeDescr` members (`canonical_name()`,
`short_name()` -- the latter made readable for templates in
`.xo-backlog/xo-reflect/issues/03`). `_name_` is unchanged: it names
variables (a member entry's member name), with an object's `_name_` a
printer-chosen label.

## Before

`_type_` held the canonical name, written two ways:

- the generic struct printer, `tp.td()->canonical_name()`
  (`xo-printjson/src/printjson/PrintJson.cpp`);
- 13 hand-written printers, `type_name<X>()` -- 3 in PrintJson.cpp
  (ObjectSlot, RootSet, AllocFlywheel), 10 in xo-websock;
- JsonMembers member entries, `type_name<Declared>()`.

No short type; the introspect page derived one in JS (`short_type()`).

## After

`json::type_keys` (new, `xo-printjson/include/xo/printjson/type_keys.hpp`)
streams both keys:

```cpp
*p_os << "{" << quot("_name_") << ": " << quot("Foo") << ", " << json::type_keys(tp.td());
//  "_canonical_type_": td->canonical_name(), "_short_type_": td->short_name()
```

Every writer uses it, so they cannot drift:

- printers with a `tp` forward `tp.td()` (the generic printer, ObjectSlot,
  RootSet, AllocFlywheel, and xo-websock's endpoint, sender, session,
  Webserver, session table, UrlRouter, router, subscription);
- `WebsocketSinkImpl::print_json` / `WebsocketSink::print_json` (Displayable
  members, no `tp`) forward `Reflect::require<X>()`;
- JsonMembers entries: `declared_of<Declared>()` (replaces `metatype_of<T>`)
  forwards `Reflect::require<Declared>()`. A C++ reference has no TypeDescr:
  its names are what one's would be -- `type_name<T>()` and
  `TypeDescrBase::make_short_name()` of it -- its metatype pointer, as before.

No writer of the old key remains:

```bash
grep -rnE '"_type_"|\\"_type_\\"|quot\("_type_"\)' --include=*.cpp --include=*.hpp xo-*/ | grep -v '/\.build/'
```

Introspect page (`xo-websock/example/introspect`): boxes and member rows read
`_canonical_type_` (source links, menu, tooltips); member rows display
`_short_type_`. Its JS `short_type()` / `drop_default_args()` are gone; a
nested object's value label is its `_name_` as printed.

No Python consumer of the key:
`grep -rn "_type_" --include=*.py xo-*/ | grep -v '/\.build/' | grep -v _metatype_` -- only unrelated
xo-cmake hits.

## Verified

- ctest 49/49. Tests that matched the key now build both from
  `type_name<T>()` + `make_short_name()` (PrintJson.test, JsonMembers.test,
  flywheel_frame.test) or assert both (`_short_type_` added in
  Webserver.test: `WebserverImpl`, `atomic<int>`, `UrlRouter`;
  WebserverLive.test: `WsSessionSender<..`; WsSessionRouter.test:
  `Subscription`).
- the 11 introspect browser tests (scratch; ticket xo-websock/13) pass; row
  expectations now read `rp<..>` (xo-reflect/03).
- in headless chrome, a real snapshot (`--src-tree`): every object / member
  entry with `_canonical_type_` has `_short_type_`; 20 of 40 distinct types
  resolve to a source link, all 20 pages fetch. The 20 without a link are
  std / builtin types, pointer / reference / template-instance types, and a
  `.cpp`-local type -- not compared against the count before this change.
- `xo-build --sweep -j 8`: 73 attempted, 73 ok (build); 47 ok + 26 with no
  tests (utest); `--sweep ok (build and utest)`.
