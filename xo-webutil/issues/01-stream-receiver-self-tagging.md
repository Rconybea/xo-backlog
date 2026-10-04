# 01 — StreamReceiver is SelfTagging; the introspect page draws receivers

Status: implemented 2026-10-03, umbrella not yet committed
Type: feature

RC: add StreamReceiver to the introspect diagram (it was not drawn: the
endpoint printer wrote `receiver_` only as a ref, and nothing printed the
receiver). RC: make StreamReceiver inherit SelfTagging instead of Refcount;
derived classes add a static `reflect_self(TypeDescrTable*)`, called from
their subsystem's appcx setup. Test receivers: `self_tp()` only. Members of
a receiver: later.

## Considered first

Name the receiver by `typeid(*receiver)`, demangled -- no change to
StreamReceiver or subclasses. Rejected (RC) for SelfTagging: exact canonical
names (the demangler spells gcc's anonymous namespace `(anonymous
namespace)`, xo's `type_name<T>` `{anonymous}`, so source links could miss),
and a path to subclass members.

## Change

- `xo-webutil/include/xo/webutil/StreamReceiver.hpp`: `class StreamReceiver
  : public reflect::SelfTagging` (was `ref::Refcount`; SelfTagging derives
  from it -- refcounting unchanged). `self_tp()` is pure virtual: every
  subclass implements it.
- New dependency xo-webutil -> xo-reflect: `xo_dependency(${SELF_LIB}
  reflect)` (the package config's find_dependency is generated from it);
  `pkgs/xo-webutil.nix` input `xo-reflect` (was commented out). No cycle:
  `xo-deps --why=xo-reflect:xo-webutil` -> no path.
- Subclasses (`grep -rn ': public StreamReceiver' --include=*.cpp --include=*.hpp xo-*/`):
  the example's `IntrospectReceiver` -- `self_tp()`, `reflect_self(table)`
  (no members: `websrv_` printed generically would nest the whole Webserver
  inside its own endpoint), called beside `IntrospectSnapshot::reflect_self`
  with `app_cx.cx<S_reflect_tag>().type_table()`; the four in xo-websock's
  utests (`RecordingReceiver`, `ThrowingReceiver`, `FrameReceiver`,
  `BoxReceiver`) -- `self_tp()` only.
- xo-websock's endpoint printer writes `"receiver": null` or `{_name_,
  _canonical_type_, _short_type_, id, refcount}` from `self_tp().td()`; the
  refcount read BEFORE `self_tp()` (the TaggedRcptr it returns holds one
  more). Its `id` is the one the `receiver_` member's ref writes.
- Introspect page: a `receiver` box, owned by its endpoint (hidden by
  default, shown by the endpoint's triangle / menu), labelled by short type,
  pale teal, in the legend; the `receiver_` row's ▾ (→) and edge.

## Verified

- `utest.websock` "webserver-json-names-each-receiver" (new): http
  endpoint's receiver null; the stream endpoint's named `NamedReceiver`,
  canonical = `type_name<NamedReceiver>()`, refcount 2 (endpoint + test),
  `receiver_`'s ref = its id. ctest 49/49.
- `xo-build --sweep -j 8`: 73 attempted, 73 ok (build); 47 ok + 26 with no
  tests (utest); `--sweep ok`.
- Headless chrome (`receiver.mjs`, new): `/introspect`'s receiver is
  `IntrospectReceiver` (`xo::web::IntrospectReceiver`), other endpoints
  none; hidden by default, shown by its endpoint's triangle; the receiver_
  row reads ▾ (→) with an edge; pale teal, in the legend; its menu's Open
  source enabled (exact type in the source map). All 15 browser tests pass
  (Show all now 12 boxes; /introspect's receiver_ row now a drawn ref).

## Open

- nix: `nix-build ci-nxfs.nix -A xo-webutil` fails -- not here, in
  xo-callback: its CMake has required refcnt since `172ecfd3` (2026-09-27,
  `xo_headeronly_dependency(${SELF_LIB} refcnt)`) but `pkgs/xo-callback.nix`
  still has `xo-refcnt` commented out:
  `nix-build ci-nxfs.nix -A xo-callback --no-out-link` -> "Could not find a
  package configuration file provided by refcnt". Pre-existing; blocks
  checking xo-webutil's nix build.
- `xo-cmake/etc/xo/subsystem-edges`: the configure's graph
  (`.build/subsystem-edges`) has three edges the committed list lacks:
  `xo-reflect xo-webutil` (this change), `xo-refcnt xo-callback` (the
  above), `xo-pyreflect xo-pywebsock` (earlier). subsystem-list already
  orders each pair correctly. Publish: `./reconfigure
  --capture-subsystem-edges`.
