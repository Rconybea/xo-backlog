# 04 — printjson accepts a fomo object directly

Status: done 2026-09-12
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

xo-printjson gains a LIBRARY dependency on xo-reflectable2. A test-only one
already exists as of `02`, in `xo-printjson/utest/CMakeLists.txt` and in
`pkgs/xo-printjson.nix` under `doCheck` -- so the edge to add here is the one
in `src/printjson/CMakeLists.txt`, and both nix inputs then collapse to one.
printjson does not depend on xo-facet today:

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

## Outcome (2026-09-12)

Done. Both entry points, plus the library edge; xo-reflect still unmodified.

### Templated on the facet, not fixed to AReflectable

The ticket said `print_obj(obj<AReflectable>, ostream*)`. Changed after checking
what callers actually hold, which is never that:

```bash
grep -n 'obj<AGCObject>' xo-object2/include/xo/object2/DList.hpp
grep -n 'DObjectHandle<' xo-pyobject2/src/pyobject2/pyobject2.cpp
#   DObjectHandle<APrintable, DFloat>
```

Fixing the signature to `obj<AReflectable>` would make every caller rotate
through FacetRegistry by hand -- work of exactly the kind the entry point
exists to remove. So:

```cpp
template <typename AFacet, typename DRepr>
void print_obj(xo::facet::obj<AFacet, DRepr> x, std::ostream * p_os) const;
```

TWO template parameters, not one: `obj<AFacet>` means
`obj<AFacet, DVariantPlaceholder>` and will not match a typed fop.

### Fast path (RC), and what it costs

`if constexpr (std::is_same_v<AFacet, AReflectable>)` goes straight to
`x.self_tp()`: such an object already carries an AReflectable implementation in
its iface, so no FacetRegistry probe and no pointer hop.

**It needs a null guard.** An EMPTY `obj<AReflectable>` carries
`IReflectable_Any`, whose `self_tp()` calls `_fatal()` -> `std::terminate()`.
Guarded by `if (x.data())`, falling through to the generic path, which renders
`{}` -- so the two agree on empty. Pinned by its own test.

The two paths are NOT equivalent in one respect, which is what makes the fast
path testable at all: comparing their OUTPUT cannot show it is taken, since
agreeing is the point. They differ in that `self_tp()` needs no registry entry
while the rotation does, so the discriminating test uses a representation that
has the `FacetImplementation` mapping and deliberately NO `register_impl<>()`:
the fast path prints it, the generic route throws.

Falsified by replacing the condition with `if constexpr (false)`: that test goes
red with `DRepr.tname _%sentinel%_`, the unregistered type's name.

### Validation entry point

`validate_tp(TaggedPtr)` plus `validate_obj`, body a no-op visitor over
`TaggedPtr::visit_tree_preorder` -- a reuse of reflect's walker, not a second
traversal. The partial-output problem it solves is pinned both ways in one test:
`validate_obj` on a bad graph throws having written nothing, while `print_obj`
on the same graph throws with a NON-empty stream.

### Edges

The library edge `printjson -> xo_reflectable2` replaces `02`'s test-only one,
and printjson's Config.cmake.in is already the generated
`@XO_FIND_DEPENDENCY_BLOCK@` form, so one `xo_dependency` line covers cmake and
the exported config alike. In nix that moves xo-reflectable2 from `doCheck`
inputs to `propagatedBuildInputs`.

The utest keeps a test-only xo-printable2 dependency, for the
printable-but-not-reflectable representation.

**Verified:** 19 assertions in 12 cases; `nix-build ci.nix -A xo-printjson` and
`-A xo-pyprintjson` green, check phase running.
