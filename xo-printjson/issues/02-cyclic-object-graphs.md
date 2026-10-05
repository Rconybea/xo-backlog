# 02 — printjson does not terminate on a cyclic object graph

Status: done 2026-10-05 -- umbrella `c14e166e`, `a6daeff6`, `19863049`, `e7c6a9b6` (follow-ups below)
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

### Identity map, decided 2026-10-04 (for step 3)

`JsonPrintState` must answer two questions:
1. Has this object been printed already in this print? If so, emit
   `{"_ref_": id}`.
2. What is its id? Both a full print (`"_id_"`) and an explicit
   `member_ref` need it, and the ref can come first.

Two facts constrain the design:
- **Distinct objects can share an address.** A struct and its first
  by-value member do, and both print as json objects.
- **A `member_ref` can know only an address.** The websock printers name a
  target by `dynamic_cast<void const *>(p)`, its most-derived address,
  through a base-class pointer (5 sites), so they cannot say what type it
  will print as.

```cpp
struct ObjectEntry {
    std::uint32_t id_;   // "_id_" / "_ref_": 1, 2, 3 .. in order of first mention
    TypeId type_;        // the type it printed as; invalid while only referred to
    bool printed_;       // printed in full yet?
};
std::unordered_map<void const *, ObjectEntry> objects_;   // key: address
std::uint32_t next_id_ = 1;
```

- **Printing an object** looks up its address:
  - absent: add an entry (new id, its type, printed), and print in full
    with `_id_`;
  - present, printed, same type: a revisit, so emit `{"_ref_": id}`;
  - present but only referred to: claim the entry (record the type, mark
    it printed) and print in full, under the id the ref already used;
  - present, printed, a different type: an inner subobject at its
    parent's address. Print it in full with no `_id_` and no dedupe. A
    ref names a whole object, so nothing refers to it; a cycle through it
    still meets the depth limit.
- **`member_ref(p)`** adds an entry (new id, not printed) if there is
  none, and emits `{"_ref_": id}`.

RC's decisions:
- **(a) Ids are per-print sequence numbers, not addresses.** Output becomes
  deterministic, so tests can pin exact strings and prints of an unchanged
  graph diff cleanly, and addresses stop leaking into the json. introspect
  joins ids only within one snapshot (it rebuilds `box_of_id` on every
  draw). Ids become json numbers, so introspect's id keys change type;
  `json_id()` goes.
- **(b) The key is the address alone.** The type is recorded, and the
  first object printed at an address owns the entry. Keying on (address,
  type) would break `member_ref` for a target not yet printed, since the
  ref site knows only a base-class view of it.
- **(c) The container is `std::unordered_map` for now, behind the
  `JsonPrintState` interface.** The intended container is xo-arena's
  `DArenaHashMap`, freed whole when the print ends, once a pool of
  temporary arenas amortizes the cost of mapping an arena per print
  (introspect prints on every tick). That is a follow-up, not part of
  step 3.

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

### Step 2 done, 2026-10-04 -- umbrella `19863049`

- `JsonPrintState::print` counts how deeply calls nest. An RAII
  `DepthScope` decrements the count on the way out, including when a
  printer throws. A call that would go past `max_depth()` reaches
  `abort_too_deep`: it prints a diagnosis (the limit and the type being
  printed), then `xo::print_backtrace`, then the diagnosis again below
  the backtrace, and aborts.

  The message says "cyclic object graph, or graph nested deeper than this
  limit". Until step 3, every cycle reaches the limit; sharpen the message
  then. RC rewrote it with `xo::pp::tostr`, not `std::string` `operator+`:
  a standing preference.
- The limit:
  - `PrintJson::c_default_max_depth = 1000`, with
    `max_depth()` / `assign_max_depth()` on `PrintJson`;
  - `PrintJsonConfig::max_depth_` and `with_max_depth()`, which
    `PrintJsonAppcx` applies to the singleton.

  Depth counts `JsonPrintState::print` calls in progress, so a struct
  holding a pointer to a struct nests two deep.
- Why 1000: a temporary probe, deleted afterwards, measured about 208
  bytes of stack per level in the debug build (416 per linked node, which
  is 2 levels). So 1000 levels is about 210 KB: 2.4x headroom under
  macOS's 512 KB secondary-thread stack (websock serves on one), and 40x
  under Linux's 8 MB.
- `PrintJson`'s class comment states the contract so far: the depth abort,
  and that a printer recurses only through its state.
- **The plan's ctest `WILL_FAIL` does not work.** Measured in a throwaway
  project (cmake 3.31): ctest reports an aborting test as "Subprocess
  aborted", a failure, under both `WILL_FAIL` and
  `PASS_REGULAR_EXPRESSION`. So the death tests fork, inside Catch2
  (`utest/PrintJsonCycle.test.cpp`, `[cycle]`):
  - the child restores the default `SIGABRT` handler (otherwise Catch2's
    own handler reports a failed test from the child) and sends its
    stderr into a pipe;
  - the parent reads the pipe dry before `waitpid`, since a backtrace
    can outgrow the pipe buffer;
  - the parent then checks for `SIGABRT` and the diagnosis.
- Tests:
  - a two-node chain within a limit of 4, exact output;
  - the same chain under a limit of 3, which aborts;
  - a two-node cycle at the default limit, which aborts;
  - a self-loop, which aborts.

  The cycle tests' expectations change in step 3, when cycles print as
  `_ref_`. Each death test takes about 1.5 s, nearly all of it the
  backtrace of about 2000 frames.
- Checked: ctest 49 / 49; `xo-build --sweep`: 73 subsystems build, and
  every subsystem's tests pass.

### Step 3 done, 2026-10-05 -- umbrella `e7c6a9b6`

Each object prints once. As decided above, with these specifics:

- **`JsonPrintState`** owns the identity map:
  - `print` (identity on) and `print_value` (identity off), both through
    `print_node`;
  - `print_ref` (`{"_ref_": n}`, the id made on first mention);
  - `open_object(name, td)` and `open_object(tp)`;
  - for an object a printer writes inline itself: `is_printed(p)` and
    `open_object_at(p, name, td)`. Only the endpoint's receiver needs
    this.

  `print_node` checks identity before it dispatches. A value's printer
  marks it an object with `JsonPrinter::prints_object()`, default true.
  The scalar, string and array printers, and `JsonPrinter_TaggedPtr`
  (which delegates), say false. With no printer, a struct is an object.
  `open_object` used twice, or outside `print_json`, aborts with a
  diagnosis.
- **`JsonObject`** (new `JsonObject.hpp` / `.cpp`) is the writer:
  - `child(k, tp)`: part of the object, identity on;
  - `key(k, v)`: a computed value, identity off;
  - `key_ref(k, p)`;
  - `key_open(k)`: the stream, for a value the printer writes itself;
  - `members()` and `close()`.

  `print_generic_struct` uses it. Its destructor, like `JsonMembers`',
  asserts it closed only when no exception is unwinding: a `validate_*`
  test throws mid-object.
- **`JsonMembers`**: `member` has identity on, `member_as` off, and
  `member_ref` / `member_refs` / `member_ref_map` go through
  `print_ref`. `json_id()` is gone, from printjson and websock.
- **`validate_tp`** runs the print's own traversal into a stream with no
  buffer. It was reflect's `visit_tree_preorder`, which recursed forever
  on a cycle. Now it reaches what `print_tp` reaches, each object once.
- **All object printers use the writer**: xo-printjson's ObjectSlot,
  RootSet and Flywheel, and websock's 10. Every hand-written `"id"` and
  `{"ref": ..}` is gone.
- **`visit_pools` hands over a `MemorySizeInfo` built on its own stack,
  so Flywheel prints each through `print_value`.** Otherwise the next
  pool, at the same stack address, would print as a ref to the first. Any
  visitor that hands over temporaries needs the same.

#### Refs name the address the target prints at

Correcting the design note above, which had `static_cast` in place of
`dynamic_cast` at all 5 sites. The rule is: a ref uses the address of the
`TaggedPtr` its target prints through. So it depends on the site:
- the **sender** prints as `WsSessionSenderImpl`, its most-derived type, so
  refs through `rp<WsSender>` keep `dynamic_cast<void const *>`. The
  sink's top-level `"sender"` used the plain base pointer and now uses the
  same cast;
- the **sink** prints through its `WebsocketSink` view, so the
  subscription's `sink_` ref is now the plain `WebsocketSink const *`. It
  had been a `dynamic_cast`: the one real mismatch;
- the **receiver**, written inline by the endpoint, uses one most-derived
  address for both the object and the `receiver_` ref.

A scratch browser check over a live introspect snapshot found 31 refs,
each resolving to one of 23 ids, and no id printed twice.
WsSessionRouter.test.cpp now prints a subscription and its endpoint
through one `JsonPrintState`, and requires the endpoint's `_id_` to
equal the subscription's `_ref_`.

#### Shared objects: placement made explicit

`pjson_` (an `rp<PrintJson>`) is held by the server, every session router
and every sink. Under first-encounter placement it printed in full
inside the first sink, because the server writes its endpoints and
sessions before its own `_members_`, and the router then drew an edge
into the sink box.

RC chose explicit placement: the routers and sinks write
`member_ref<rp<PrintJson>>`, and the server, its owner, prints it in full.
Each router and sink now has a "shares" edge into the server box. The
rejected options were reordering the server's keys, which is implicit,
and printing PrintJson as a scalar, which is a special case.

#### Consumers and tests

- **introspect.js:** `_ref_` and `_id_`; ids are numbers. "Copy id" copies
  the number.
- **printjson tests:** `_id_` in pinned output. `PrintJsonCycle.test.cpp`
  covers:
  - a cycle and a self-loop, as exact `_ref_` output;
  - a diamond;
  - a chain of 40 diamonds: 41 ids and 40 refs, where printing every path
    would take 2^40;
  - a ref before its print, claimed by the later print;
  - a first member sharing its parent's address: in full, no `_id_`;
  - `validate_tp` terminating on a cycle;
  - the depth-limit death test.
- **Other tests:** websock and object2 tests moved to `_id_` / `_ref_`, and
  the flywheel frame pins `_id_` 1 (the frame) and 2 (its root set), with
  none on the pools.
- **Browser tests:** rows and expand changed for numeric ids and the
  `_ref_` key. router_expand and urlrouter_expand now expect the
  `pjson_` edge into the server.
- **Checked:**
  - ctest: 49 / 49;
  - the 19 introspect browser tests;
  - `xo-build --sweep`: 73 subsystems build, and every subsystem's tests
    pass.

### Follow-ups (not done here)

- **DArenaHashMap for the identity map,** drawn from a pool of temporary
  arenas: RC's intent, see "Identity map".
- **`JsonPrintState`'s public constructor.** A printer could make a fresh
  state and escape both identity and the depth limit. Neither the
  re-entry guard nor the depth limit catches that. Tests use the
  constructor.
- **`XO_PRINTJSON_REENTRY_CHECK`** is defined for now, to be turned off
  later (RC).
- **The collapse toward a reflection-driven printer.** The websock
  printers still repeat members as hand-picked top-level keys (the sink's
  `refcount` / `stream` / `sender`, ..). Retire them when the bespoke
  printers go.
- **The ref-resolution check** (`refs_resolve.mjs`) and the other browser
  tests live in a session scratchpad, not the repo.

**Done when** (unchanged, made concrete): a cyclic graph prints finite
output, with each object once; `PrintJson.hpp` states the contract; tests
build real cycles and a diamond, and pin the output.
