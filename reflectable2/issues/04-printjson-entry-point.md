# 04 — printjson accepts a fomo object directly

Status: open
Type: feature
Milestone: reflectable2

`PrintJson` has one entry point for opt-in reflectable objects, and it takes the
non-fomo capability:

```bash
grep -n 'print_obj\|SelfTagging' xo-printjson/include/xo/printjson/PrintJson.hpp
```

Add the fomo sibling, so a caller holding `obj<AReflectable>` need not build a
`TaggedPtr` by hand.

## Shape

`void print_obj(obj<AReflectable> x, std::ostream * p_os) const`, beside the
existing `print_obj(rp<SelfTagging> const &, std::ostream *)`. Body is the same
one line in spirit: get the object's `TaggedPtr`, hand it to `print_tp()`.

Only the entry point is new. Nested fomo members already work after `03`,
through the struct-member path — so if this ticket seems to require changes to
`print_aux` or the `Metatype` switch, something in `02`/`03` is wrong and that
is the thing to fix.

xo-printjson gains a dependency on xo-reflectable2. It does not depend on
xo-facet today:

```bash
xo-deps --why=xo-printjson:xo-facet -q; echo "exit=$?"     # expect 1 before this ticket
grep -n '^xo-printjson$' xo-cmake/etc/xo/subsystem-list    # well above reflectable2
```

**Files:**
- Modify: `xo-printjson/include/xo/printjson/PrintJson.hpp`,
  `src/printjson/PrintJson.cpp`, `src/printjson/CMakeLists.txt`
- Test: `xo-printjson/utest/`

**Done when:**
- `print_obj(obj<AReflectable>, &os)` emits the same JSON as passing the
  equivalent hand-built `TaggedPtr` to `print_tp()`
- a struct with an erased fomo member round-trips to JSON with no printjson
  change beyond this entry point
