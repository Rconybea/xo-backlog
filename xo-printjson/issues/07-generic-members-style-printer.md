# 07 -- the generic struct printer writes members-style

Status: done 2026-10-05 -- umbrella `97730b40`
Type: feature
Milestone: reflection-driven-json
Blocked by: `.xo-backlog/xo-printjson/issues/06`

Change `print_generic_struct` (`JsonPrintState.cpp`) from the flat format,
where members are top-level keys, to members-style:
`{"_name_", type keys, "_id_", "_members_": [...]}`, built from reflection as
in `issues/06`.

Placement is first encounter (decided in the milestone):
- a by-value member prints inline (`child`, identity on);
- pointers and `rp` print through `state.print`, so print-once places them.

The flat format has a known consumer: the flywheel frame's MemorySizeInfo
and RootSet keys are a declared wire contract with a browser outside this
repo (`xo-object2/utest/flywheel_frame.test.cpp`,
`xo-printjson/src/printjson/PrintJson.cpp`, "The keys are a wire contract
with a browser"). Either keep flat as an explicit, recorded exception for
those types until their consumer moves, or upgrade them together. Check
`xo-pyprintjson` and the other generic-struct consumers before switching.

**Done when:** a reflected struct with no registered printer prints
members-style; any flat output that remains is an explicit choice and is
recorded.

## Done, 2026-10-05 -- umbrella `97730b40`

**The change.** `print_generic_struct` (`JsonPrintState.cpp`) is now
`open_object(tp)`, then `members().reflected_members(tp).end()`, then
`close()`. Every reflected struct with no registered printer prints
members-style, under its reflected names, with no name suffix.
- An empty struct prints `"_members_": []`, so the shape never varies.
- A member that cannot print becomes an `_error_` entry. The flat printer
  wrote `<error-json-printer-not-found ..>` inline, which is not valid
  json.

**No flat exception.** RC (2026-10-05): MemorySizeInfo adopts `_members_`,
because nothing outside the repo relies on the flywheel frame's format.
`flywheel_frame.test.cpp` pins the new frame:
- a `pool(..)` builder makes each expected pool, taking declared types from
  `decltype` of the real fields;
- the redaction regexes now match a member entry's `_value_`;
- its comments, and `PrintJson.cpp`'s, no longer claim a browser wire
  contract.

**Consumers**, found by a survey of reflected types with no printer:

| Type | Consumer | Change |
|---|---|---|
| IntrospectSnapshot `{server}` | introspect.js; browser tests header, sink, noticker, receiver, expand | page helper `member_value(obj, name)`; the tests call it |
| MemorySizeInfo | flywheel frame test | expectations, as above |
| UpxEvent, KalmanFilterState(Ext) | `xo-websock/utest/mount-origin/ex_websock.js` (`tm`, `upx`, `tk`, `x`, `P`) | helper `mv(obj, name)`. Syntax-checked only: its page server (`websock_utest_main`) is disabled |
| HoldsServer (test) | Webserver.test.cpp | `member_value` helper |
| printjson test structs | PrintJson / PrintJsonCycle / JsonMembers tests | expectations rebuilt members-style |

The survey found no json consumer for the other types that print
generically: ExpProcess, BrownianMotion, KalmanFilterInput and
KalmanFilterTransition, DObjectEvent / DTypedEvent, VsmStackFrame,
LocalEnv.

**Golden snapshot:** one line. The server's `PrintJson`, reflected with no
members, gains `"_members_": []`.

Checked: ctest 49 / 49; the 20 browser tests; `xo-build --sweep`.

**Left for `xo-websock/issues/16`:** the websock types reflect with
`REFLECT_MEMBER` (stripped names), and their printers pass
`name_suffix = "_"` to keep the C++ names. The generic printer passes no
suffix, so retiring those printers either renames their members
(`port_` -> `port`) or makes the suffix a `PrintJson` option.
