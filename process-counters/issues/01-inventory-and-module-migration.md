# 01 — inventory of process-wide counters, and whether they belong in a Module

Status: open
Type: task

Raised by RC 2026-08-10, from the phase-C printer conversions in
`.xo-backlog/xo-printable2/issues/01-aprintable-pretty-ppsink.md`.

Several counters in xo are **process-wide function-local statics**. They are
handed out in first-use order, so their values depend on what else the process
did first — including, in a test binary, which other tests ran. That is why
`xo-expression2/utest/printable_render.test.cpp` now carries **three separate
scrubbers**, one per counter, purely to keep pinned renderings stable.

RC's proposal: consider moving them into a **Module** — an instantiable context
object owning the state, of which `xo::scm::Schematika`
(`xo-interpreter/include/xo/interpreter/Schematika.hpp:17`, "schematika
interpreter state", `Schematika::make(const Config &)`) is the existing
exemplar. **A unit test would then start from a known state rather than
inheriting a non-zero process counter.**

## The inventory

Measured 2026-08-10:

```bash
grep -rnE "static +(std::)?(atomic<)?(u?int(32|64)_t|unsigned|size_t|long)[a-z_ <>]* +s_[a-z_]*(count|counter|id|seq|next|serial|gen)" \
  --include=*.hpp --include=*.cpp xo-*/ | grep -v '/\.build/' | grep -v '/utest/'
```

| # | counter | file | numbers | observable as | atomic? |
|---|---|---|---|---|---|
| 1 | `s_next_id` | `xo-reflectutil/.../typeseq.hpp:51` | type registration | `:value.tseq 9` | no |
| 2 | `s_next_id` | `xo-reflect/.../TypeDescr.hpp:41` | reflection order | `:id 42` (TypeId) | no |
| 3 | `s_counter` | `xo-expression2/src/expression2/TypeRef.cpp:86` | `generate_unique()` type-variable names | `:id "if:12"` | no |
| 4 | `s_counter` | `xo-expression/src/expression/typeinf/type_ref.cpp:29` | as (3), legacy expression stack | type-var names | no |
| 5 | `s_counter` | `xo-expression/src/expression/Variable.cpp:16` | `Variable::gensym()` | generated symbol names | no |
| 6 | `s_counter` | `xo-stringtable2/src/stringtable2/StringTable.cpp:82` | `StringTable::gensym()` | generated symbol names | no |
| 7 | `s_ppconfig_seq` | xo-indentlog2 | scratch-arena naming | arena names in diagnostics | **no, and not even `static`** — see below |

(7) is already ticketed separately as
`.xo-backlog/xo-indentlog2/issues/06`, which found it had lost both `static` and
atomicity. It is listed here because it is the same species and any policy this
ticket lands on should cover it.

**Unverified:** the grep matches a naming convention (`s_*`), so it may miss
counters named otherwise, and it excludes `utest/`. Treat the table as a
starting point, not a closed set — and prefer re-running the grep to trusting
the rows.

## Why it matters, concretely

**1. It leaks into output, and the tests have to work around it.** Three
scrubbers, in one test file, each for a different counter:

| scrubber | counter | pattern scrubbed |
|---|---|---|
| `scrub_type_id` | (2) | `:id 42` |
| `scrub_tseq` | (1) | `:value.tseq 9` |
| `scrub_typevar` | (3) | `:id "if:12"` |

Every one of them exists because the value cannot be pinned. A rendering test
that *wants* to assert an identifier cannot, and each new printer that shows one
needs another scrubber or another exclusion.

**2. Tests are order-dependent in a way that is invisible until it bites.**
`scrub_tseq`'s own comment records that typeseq is "stable today — DInteger is 9
and DFloat 10 on every run — but it is registration order". It is stable because
registration happens in fixed setup code, not because anything guarantees it.
Adding a D-type to `SetupObject2::register_facets` moves every number after it.

**3. None of them are atomic.** Not currently a bug — they are touched during
single-threaded setup — but it is a latent one, and (7) shows the invariant is
already not being maintained by hand.

## What a Module would change

`Schematika` is the shape: a `Config`, a `make()`, an instance owning the state
that a program used to keep in globals. Moving a counter into a Module means:

- a unit test constructs the Module and gets counters starting from 0 —
  **deterministic identifiers, so renderings could be pinned exactly and the
  scrubbers deleted**
- two Modules in one process are independent, which is a prerequisite for ever
  running xo tests in parallel within a binary
- lifetime is explicit, so "who owns the type registry" becomes answerable

## What needs deciding, and the hard part

**Not all seven are the same kind of thing**, and this is the substance of the
ticket rather than a detail:

- **(5), (6) gensym counters** look easy: they generate names for one
  compilation/session, and a `Module` (or the `StringTable` itself, which is
  already an instance) is the obvious owner. `StringTable::gensym` is already a
  member function of an instance — its counter is `static` inside it, which
  looks more like an oversight than a design.
- **(3), (4) type-variable names** belong to a type-inference run, so the owner
  is plausibly the inference context rather than a global Module.
- **(1), (2) typeseq / TypeId are the hard ones.** They identify types
  *program-wide* and are used as keys into registries that are themselves
  process-wide (`FacetRegistry`, `TypeRegistry`, `CollectorTypeRegistry`).
  Making them per-Module means those registries become per-Module too, which is
  a much larger change than renumbering a counter — and `typeseq::id<T>()` is a
  template whose whole point is to be callable from anywhere without a context
  argument. **Do not assume the answer is the same for all seven.**

Worth considering as a cheaper intermediate for (1) and (2): a *reset* entry
point for test setup, rather than full per-Module ownership. That would give
tests a known starting state — the stated goal — without re-plumbing the
registries. It has its own hazard (a reset while live objects hold old ids), so
it needs the same scrutiny, but it is a much smaller change and may capture most
of the value.

## Suggested approach

1. Re-run the grep; treat the table above as stale on sight.
2. Classify each counter by *who should own it* before designing anything —
   the three groups above are a hypothesis, not a conclusion.
3. Start with (5)/(6), the gensym pair: smallest, and they prove or disprove the
   ergonomics of threading a Module through call sites.
4. Only then decide whether (1)/(2) want per-Module ownership, a reset hook, or
   to stay as they are.

Success is measurable: **the number of scrubbers in
`xo-expression2/utest/printable_render.test.cpp` should go down**, and the
renderings they currently blank out should become pinnable exactly.

**Files:**
- the seven sites in the table
- `xo-interpreter/include/xo/interpreter/Schematika.hpp` — the Module exemplar
- `xo-expression2/utest/printable_render.test.cpp` — the three scrubbers, and
  the evidence for the whole ticket
- `.xo-backlog/xo-indentlog2/issues/06` — counter (7), already ticketed

**Done when:**
- the inventory is confirmed complete by something better than a name-pattern grep
- each counter has a recorded decision: move to a Module, add a reset hook, or
  keep as-is with the reason
- at least the gensym pair is done, or explicitly deferred with a reason

## Notes

**This is a design question, not a defect report.** Nothing is broken today; the
counters work and their values are currently stable. The cost is paid in tests
that cannot assert what they would like to, and in a fragility that has so far
only been contained rather than removed.
