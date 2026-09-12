# 02 — printjson does not terminate on a cyclic object graph

Status: open
Type: bug

`PrintJson::print_aux` recurses into children with no record of what it has
already visited, so an object graph containing a cycle recurses until the stack
is exhausted:

```bash
grep -n 'visited\|seen\|depth\|cycle' xo-printjson/src/printjson/PrintJson.cpp   # no hits
grep -n 'print_aux' xo-printjson/src/printjson/PrintJson.cpp                     # the recursion
```

The generic walker in xo-reflect has the same limit, but at least states it:

```bash
sed -n '27,34p' xo-reflect/include/xo/reflect/TaggedPtr.hpp
# "require: no cycles in object graph -- undefined behavior if a cycle is present"
```

printjson does not state it anywhere.

## Not caused by fomo

Worth saying, because the first draft of this ticket got it wrong: cycles do
NOT require a collector, and are not new with faceted objects. Any two objects
holding pointers to each other form one — refcounted graphs and plain
back-pointers included. This has been true since printjson existed, and applies
equally to the legacy types it was built for.

What is true is that anything walking a graph hits it: the validation entry
point in `.xo-backlog/reflectable2/issues/04` is defeated by a cycle exactly as
printing is, since both go through the same traversal.

Representable in practice — `DList` guards its rest-chain by comment only, and
its head, like a dictionary's values, is an arbitrary object:

```bash
grep -n '_assign_rest\|assign_head' xo-object2/include/xo/object2/DList.hpp
# "Caller responsible for preserving acyclic property!"
```

## Remedy — deliberately not chosen

Left open until picked up, so the cost of each is judged against the code as it
is then. Candidates:

1. **Visited set** — track addresses emitted during one print; on revisit emit
   a back-reference or an error. Correct for arbitrary graphs. Costs a set per
   print, and forces a decision about what a revisited node renders as: JSON
   has no native reference, so this is a format question, not only a code one.
2. **Depth limit** — bound recursion, throw past it. Catches cycles and runaway
   nesting alike with no per-node bookkeeping, but the limit is arbitrary and a
   legitimately deep graph fails.
3. **Documented precondition** — state that printjson requires an acyclic
   graph, matching what reflect already says, and leave detection to the
   caller. Cheapest, and leaves the failure mode a stack overflow.

Note (1) and (2) are not exclusive: a depth limit is a cheap backstop even if a
visited set is the real answer.

**Done when:**
- a cyclic graph produces a defined outcome rather than a stack overflow
- whichever remedy is chosen, printjson STATES its precondition, which it does
  not today
- the chosen behaviour is pinned by a test that builds an actual cycle
