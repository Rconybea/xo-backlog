# 05 -- a map metatype

Status: open
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
