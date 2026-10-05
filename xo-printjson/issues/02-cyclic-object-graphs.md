# 02 — printjson does not terminate on a cyclic object graph

Status: open (design decided; steps -1, 0, 1 done -- umbrella `c14e166e`, `a6daeff6`)
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

## Remedy — candidates, as first written

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

## Design decided, 2026-10-04

RC picked the remedy. In short: every object prints once, the per-print
state passes explicitly through every printer, and a depth limit aborts as
a backstop.

Checked against the code on 2026-10-04 (umbrella `4ab4402c`), the bug
stands as diagnosed:

```bash
grep -n 'visited\|seen\|depth\|cycle' xo-printjson/src/printjson/PrintJson.cpp
# only a comment hit (line 545)
sed -n 75,98p xo-printjson/src/printjson/PrintJson.cpp
# print_generic_pointer follows an rp<T> straight into its target
```

Since this ticket was written, xo-websock avoids cycles by convention:
each object prints in full once, where it is owned, and elsewhere as
`{"ref": json_id(p)}` via `JsonMembers::member_ref`
(`xo-websock/src/websock/websock_json.cpp:8`). That holds only because
every printer there was written by hand to keep it. The generic pointer
path, and any printer that recurses on a pointer, has no such guard.

### Choices and why

- **Every object prints once, not just a back-reference on a true cycle.**
  RC: marking only true cycles leaves a DAG free to need O(2^n) output (a
  chain of diamonds). That exhausts bounded resources as surely as
  non-termination does.
- **The per-print state threads through the bespoke printers, explicitly.**
  I proposed a thread_local "print in progress" record, so that the
  `JsonPrinter` signature need not change. RC rejected it: it obscures the
  actual data dependence in the completed solution, which makes it harder
  to reason about. Design the end state first, then choose the path.
- **Depth limit: abort with a stack trace.** A printer that recurses other
  than through the framework's entry point is a bug, so reaching the limit
  is a bug too, not a condition to report in the output.
- **Every object prints its id, as `_id_`; a reference is `{"_ref_": id}`.**
  This pairs with `_name_` and the type keys. It also stays clear of member
  names: a generic struct prints its members as top-level keys, so a member
  called `ref` would look like a reference.
- **Identity covers only values printed as json objects.** Scalars and
  arrays have no braces to carry an `_id_`, and only objects can make
  printing fail to terminate. (A cycle running only through arrays would
  still hit the depth limit.)
- **One atomic change of the printer signature** across all its overrides,
  with no temporary shim for the old signature.

### End state

1. **`JsonPrintState`** (name open) holds what belongs to one top-level
   print:
   - the output `std::ostream *`;
   - the `PrintJson const *` whose printer table it dispatches through;
   - the identity map, (address, `TypeId`) -> `_id_`;
   - the current depth, and the limit.

   `JsonPrinter::print_json(TaggedPtr, JsonPrintState &)` replaces
   `(TaggedPtr, std::ostream *)`. A printer gets output and recursion only
   through the state it is given. `state.print(tp)` is the one way to
   recurse; it replaces `print_aux`, which stops being public.
2. **The public entry points** (`print`, `print_tp`, `print_obj`,
   `validate_tp`, `validate_obj`) each make one fresh state per top-level
   value, so an `_id_` is unique within one output. `validate_*` runs the
   same traversal with the same state and discards the output.
3. **Print once.**
   - `state.print` checks before it dispatches to a printer, so no printer
     can skip the check.
   - A printer declares whether it prints a json object. The default comes
     from the metatype (`mt_struct` yes); `AsStringJsonPrinter` and the
     scalar and vector printers say no.
   - The first time an object is reached, it prints in full. Every later
     time, it prints `{"_ref_": id}`.
   - The key is address plus type, since a struct and its first member
     share an address.
4. **Object writer.** An object printer opens its object through the
   state, which writes `{"_name_": .., type keys, "_id_": ..`.
   `JsonMembers` hangs off that writer, rather than taking
   `(pjson, p_os)`. The websock printers' hand-written `"id"` keys
   (11 sites) go.
5. **Placement stays explicit.** The generic printers place an object where
   the traversal first reaches it. `member_ref` keeps its role of saying
   "owned elsewhere, do not print it here". It is no longer needed for
   termination.
6. **Values with no lasting address.** `member` (an lvalue inside the
   object) takes part in identity. `member_as` (a computed value, e.g.
   `x->port_.load()`) prints with identity off: a temporary's stack
   address can be reused by another temporary within one print. What a
   temporary points to is still tracked.
7. **Depth limit.** Past the limit, `state.print` calls
   `xo::print_backtrace` (`xo-arena/include/xo/arena/backtrace.hpp`;
   printjson already depends on xo_arena) and then `std::abort()`. The
   limit lives in `PrintJsonConfig` (`xo-printjson/include/xo/printjson/cx/`).
8. **Contract.** `PrintJson.hpp` states it:
   - any graph prints in finite output;
   - each object prints once, first encounter wins, unless a printer writes
     a ref explicitly;
   - nesting deeper than the limit aborts.

   The `validate_tp` comment stops saying "a CYCLIC graph defeats both
   passes".

### Path

Each step is one umbrella commit, building and passing its tests on its own.

Steps -1 and 0 landed together in umbrella `c14e166e`.

-1. **Re-entry guard.** `print_tp` and `validate_tp` hold an `EntryGuard`
    for as long as they run. Every public entry point funnels into one of
    them: `print<T>` and both `print_obj` overloads into `print_tp`,
    `validate_obj` into `validate_tp`. If another entry point is already
    active on the thread, the guard calls `xo::print_backtrace` and then
    `std::abort()`. That means a json printer started a print within a
    print, instead of recursing via `print_aux`.
    - The counter is `thread_local`. RC chose this over a plain flag (a
      legitimate print on another thread would look like re-entry, and it
      would be a data race on a `const` entry point) and over an atomic
      owner-thread id (overlapping prints on two threads would go
      unchecked). It is an assertion only: no printing reads it.
    - It compiles only under `XO_PRINTJSON_REENTRY_CHECK`, defined
      PRIVATE on the printjson library (`src/printjson/CMakeLists.txt`).
      RC: on for now, to be turned off later. The guard lives in
      `PrintJson.cpp` and the header has no `#if`, so translation units
      cannot disagree about it.
0. **Every printer recurses via `print_aux`.** With -1 alone, three test
   binaries abort: utest.printjson, utest.stringtable2, utest.object2.
   The guard reports these five call sites, the same five that
   `grep -rn 'pjson()->print\(\|pjson()->print_tp('` finds:
   - `PrintJson.cpp`, `JsonPrinter_TaggedPtr`: `print_tp(*x, ..)`;
   - `EigenUtil.cpp`, the vector and matrix element loops: `print(..)`;
   - `SetupObject2.cpp`, `DFloatJsonPrinter`: `print(x->value(), ..)`;
   - `SetupStringtable2.cpp`, `DStringJsonPrinter`: `print(string_view(..), ..)`.

   Each becomes `print_aux(Reflect::make_tp(&v), ..)`. A temporary (a
   double, a string_view) is bound to a local first. -1 and 0 land
   together, or 0 first, so no commit has a red test.

1. **Signature.**
   - Add `JsonPrintState`.
   - Drop `JsonPrinter(PrintJson const *)`, `pjson()` and the uncalled
     `assign_pjson`: the state carries the printer table.
   - Change `print_json` to take it, in all 26 `JsonPrinter` overrides:
     - xo-printjson: 11 in `PrintJson.cpp`, plus `AsStringJsonPrinter` in
       `JsonPrinter.hpp`;
     - xo-websock: 10 (`Webserver.cpp` 4, `websock_json.cpp` 3,
       `WsSessionRouter.cpp` 2, `UrlRouter.cpp` 1);
     - xo-kalmanfilter: 2;
     - xo-object2: 1;
     - xo-stringtable2: 1.

     Also xo-websock's own virtual
     `WebsocketSink::print_json(PrintJson const &, std::ostream *)`, which
     a `websock_json.cpp` printer calls: it takes the state too.

     Count with:

     ```bash
     grep -rn --include=*.hpp --include=*.cpp -E 'print_json\s*\(\s*(xo::)?(reflect::)?TaggedPtr' . \
         | grep -v '^./.build'   # 27 lines: 26 overrides + the pure virtual
     ```
   - Retire `print_aux` from the public API. `JsonMembers` takes the state.

   The output does not change, so the existing tests pin the change.
2. **Depth limit**, with the contract's abort clause. Catch2 has no death
   tests, so the abort test would be a small driver run by ctest and
   expected to fail. Whether `WILL_FAIL` counts an abort (a signal, not an
   exit code) as the expected failure needs checking when this is built.
3. **Print once.**
   - Add the identity map, the object writer, `_id_` on every object, and
     `{"_ref_": id}`.
   - Rename `ref`/`id` to `_ref_`/`_id_` in `JsonMembers` and the websock
     printers, and in xo-websock's introspect.js and its browser tests.
   - Add tests for a two-node `rp<>` cycle, a self-loop and a diamond, plus
     the existing expected outputs updated for `_id_`.

### Step 1 done, 2026-10-04 -- umbrella `a6daeff6`

As planned, with these specifics:

- `JsonPrintState` (`JsonPrintState.hpp` / `.cpp`) holds the output and
  the printer table. It offers:
  - `p_os()`, the output;
  - `has_printer(td)`;
  - `print(tp)`. This now holds `print_aux`'s dispatch, and the generic
    pointer / vector / struct printers moved with it.

  `PrintJson::print_aux` is gone. `print_tp` makes the state, inside the
  re-entry guard. Printer lookup is a private `PrintJson::lookup_printer`,
  reached by `JsonPrintState` as a friend.
- `JsonPrinter::print_json(TaggedPtr, JsonPrintState &)`. The constructor
  argument, `pjson()`, `pjson_` and the uncalled `assign_pjson` are gone.
  `check_recover_native` and `report_internal_type_consistency_error` take
  the state. `JsonMembers(JsonPrintState &)` replaces `(pjson, p_os)`.
- The 26 overrides were rewritten by a script that scopes each change to
  one `print_json` body, then reviewed by hand. Each body changes only in:
  - `std::ostream * p_os = state.p_os();` at its top;
  - `state.print(..)` for recursion;
  - `&state` captured by its lambdas.
- `WebsocketSink::print_json(json::JsonPrintState &)`. It prints `this`,
  so it takes no `TaggedPtr`.
- Left as is, for now:
  - **The `JsonPrintState` constructor is public.** `JsonMembers.test.cpp`
    makes states directly. Nothing stops a printer making one; its doc
    says not to, and the re-entry guard does not catch it.
  - **`validate_tp` is unchanged.** It walks the graph with reflect's own
    traversal, not through printers. It joins the state in step 3, with
    identity.
- Output unchanged, checked three ways:
  - ctest: 49 / 49;
  - the 19 introspect browser tests, which pin the websock json in detail;
  - `xo-build --sweep`: 73 subsystems build, and every subsystem's tests
    pass.

**Done when** (unchanged, made concrete): a cyclic graph prints finite
output, with each object once; `PrintJson.hpp` states the contract; tests
build real cycles and a diamond, and pin the output.
