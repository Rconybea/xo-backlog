# 05 -- a map metatype

Status: done 2026-10-11 -- umbrella `339edf66`, golden `357fc94c` (reduced scope: websock adoption moved to `xo-websock/issues/15`)
Type: feature
Milestone: reflection-driven-json

xo-reflect has no associative containers. The metatypes are atomic,
pointer, vector, struct and function (`Metatype.hpp:9`), and `std::map` /
`std::unordered_map` fall to opaque `AtomicTdx`. RC (2026-10-05): add a
new metatype for them.

Design to settle here:
- the child interface (key and value per entry; keys as values or as
  TaggedPtrs);
- order: an unordered map has none, so print in sorted key order for
  deterministic output;
- how printjson renders it: a json object for string keys, else an array
  of pairs.

Meanwhile the websock maps (`UrlRouter::http_map_` / `stream_map_`,
`WsSessionTable::session_map_`) keep their custom printers:
`member_ref_map` over a sorted copy.

**Done when:** a `std::map` and a `std::unordered_map` member print through
reflection, in sorted key order, and the websock maps need no custom code.

## Resolution (2026-10-11)

**Scope reduced (RC, 2026-10-10).**  The last clause -- "the websock maps
need no custom code" -- cannot land here.  Reflecting `session_map_` as an
owning edge trips the two-owners assert (`xo-printjson/issues/08`): the
server's `sessions[]` view-model list places the sessions first.  So
`member_ref_map` stays in `UrlRouter.cpp:293-294` and `Webserver.cpp:1255`
until `xo-websock/issues/17`; adopting maps there is the rest of
`xo-websock/issues/15`.

What landed (umbrella `339edf66`):
- xo-reflect: `Metatype::mt_map`; `map/MapTdx.hpp` -- `MapTdx` with
  `key_td()`, `value_td()`, `visit_entries(object, fn)` (in key order,
  entries visited in place); `StdMapTdx<MapT>` sorts an unordered map
  with `<` when the key type supports it, else keeps iteration order.
  `EstablishTdx` for `std::map` / `std::unordered_map`.  Entries are
  owning children, like a vector's.  Tests: `utest/MapTdx.test.cpp`.
- xo-printjson: `JsonPrintState::print_map`.  A map prints with no
  `_id_`.  Keys that print as a json string or number give a json object
  (numbers quoted) -- the shape `member_ref_map` already wrote, so
  introspect's reading of `session_map_["1"]` holds; any other key type
  gives `[{"_key_": k, "_value_": v}, ...]`.  Tests:
  `utest/PrintJsonMap.test.cpp`.
- xo-pyreflect: `Metatype.map`.
- websock golden (`357fc94c`): `http_map_`, `stream_map_`,
  `session_map_` report `"_metatype_" : "map"`, not `"atomic"`; values
  unchanged.

**clang workaround.**  `StdMapTdx::c_sortable` uses a nested requirement
(`requires std::convertible_to<decltype(a < b), bool>;`), not
`{ a < b } -> std::convertible_to<bool>`.  clang 19.1's `-ast-dump`
segfaults (in `ExprRequirement::ReturnTypeRequirement::getTypeConstraint`)
dumping a FAILED compound requirement, which xo-type-src-map runs on every
TU; any TU instantiating `StdMapTdx` with a key lacking `<` lost its whole
source-map contribution.  Reproduce outside the tree:

```cpp
template <typename F, typename T> concept conv = requires (F f) { static_cast<T>(f); };
template <typename K> struct S {
    static constexpr bool b = requires (K const & a, K const & c) { { a < c } -> conv<bool>; };
};
struct U { int v; };
static_assert(!S<U>::b);
// clang++ -std=c++20 -fsyntax-only -Xclang -ast-dump -Xclang -ast-dump-filter=S x.cpp
//   => "clang frontend command failed with exit code 139"; without -ast-dump, parses clean
```
