# 07 -- the generic struct printer writes members-style

Status: open
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
