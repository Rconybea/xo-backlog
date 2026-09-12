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

### Plus a validation entry point

`03` makes a non-opted-in representation throw, and throwing mid-traversal
leaves whatever the consumer already wrote -- for printjson, a truncated
document. So also provide an entry point that WALKS without printing, and
throws on the same condition: a caller that wants all-or-nothing validates
first, then prints.

Nearly free -- reflect already has the walker, so this is a no-op visitor over
it rather than a second traversal implementation:

```bash
sed -n '27,45p' xo-reflect/include/xo/reflect/TaggedPtr.hpp   # visit_tree_preorder
```

Two properties to state where the function is declared, because neither is
visible at the call site: it costs a second full traversal, and it is only
meaningful because traversal is synchronous -- nothing may mutate the graph
between the two passes.

It does NOT make printing total. A cyclic graph defeats both passes; that is
`.xo-backlog/xo-printjson/issues/02`, and is not caused by this milestone.

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
- the validation entry point throws for a graph containing a non-opted-in
  representation, and writes nothing
- validate-then-print over a fully-opted-in graph produces the same output as
  print alone
- a struct with an erased fomo member round-trips to JSON with no printjson
  change beyond this entry point
