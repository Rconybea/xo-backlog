# 01 — APrintable::pretty(ppindentinfo) -> pretty(PpSink&), across the facet cluster

Status: open
Type: task
Milestone: ppsink-migration
Progress: grep -rl 'PHASE B STUB' --include=*.hpp xo-*/ | grep -v '/\.build/' | wc -l

The last and largest piece of the indentlog->ppsink migration: the facet cluster
rooted at `xo-printable2`, whose facet method is still the legacy two-pass
protocol.

```cpp
/* generated from xo-printable2/idl/Printable.json5 */
bool pretty(Copaque data, const ppindentinfo & ppii) const;   /* legacy  */
void pretty(Copaque data, PpSink & sink) const;               /* target  */
```

**Filed 2026-08-09 because it did not exist.** The milestone described this
work but no ticket carried it, so `xo-sdlc --milestone=ppsink-migration` read
7/11 while the dominant remaining effort was invisible to the query. Progress
being derived from tickets is right until something has no ticket — then it
reads as done. Worth watching for elsewhere.

## Scope: "233 files" measures generator output, not work

The figure carried by the milestone and `xo-ppsink/issues/02` is a file count
over the whole cluster. **Roughly half of it is generated** — the facet
scaffolding (`AFoo`, `IFoo_Any`, `IFoo_Xfer`, `RFoo`, `Foo.hpp`) is produced by
`xo-facet/codegen/genfacet` from jinja2 templates plus an IDL file, and carries
a `Generated automagically` banner.

Measured 2026-08-09:

```bash
# generated files tree-wide, and how many touch this protocol
gen=$(grep -rl 'Generated automagically' xo-*/ --include=*.hpp --include=*.cpp | grep -v '/\.build/')
echo "$gen" | wc -l                                              # 362
echo "$gen" | xargs grep -l 'ppindentinfo\|ppdetail' | wc -l      # 114

# hand-written files declaring or defining pretty(ppindentinfo)
all=$(grep -rln 'pretty *(.*ppindentinfo\|ppindentinfo & *ppii' xo-*/ \
        --include=*.hpp --include=*.cpp | grep -v '/\.build/' | grep -v '^xo-indentlog/')
comm -23 <(echo "$all" | sort) <(echo "$gen" | sort) | wc -l      # 111
```

| | count |
|---|---|
| hand-written implementations | **111 files** — `DFoo.hpp`/`DFoo.cpp` pairs, so ~55 types |
| of which xo-reader2 | 48 |
| xo-expression2 / xo-interpreter2 / xo-object2 | 24 / 16 / 15 |
| xo-stringtable2 / others | 4 / 4 |
| generated, regenerate themselves | ~114 |

Non-`D` exceptions worth knowing about: `TypeRef`, `ParserStack`,
`ParserResult`.

Also note the outside-the-cluster group has collapsed since the milestone was
written: it recorded 65 files outside, of which xo-expression 28 and xo-reader
24; those landed with `.xo-backlog/xo-expression/issues/01`. It is now **15**,
so "what remains is the facet cluster" — false on 2026-08-08, per
`xo-ppsink/issues/02` — is true again as of 2026-08-09. 233 inside, 15 outside.

## Plan (RC, 2026-08-09): rename first, then add

The point of this sequence is that the expensive-looking step is mechanical and
the tree stays green throughout.

**Phase A0 — resync generated code with the current templates. DO THIS FIRST,
as its own commit.** Regenerating with an *unchanged* IDL was not a no-op: the
committed generated code predated the templates, so a rename would have carried
unrelated drift along with it and a failure could have been either. Done
2026-08-09; see the A0 section below for what it found.

**Phase A — rename, no new behaviour.** In `xo-printable2/idl/Printable.json5`,
rename the `pretty` const_method to `pretty_deprecated`. Rebuild the genfacet
targets; every generated file follows. Hand edits are then only the renames in
the 111 `DFoo.*pp` implementations, plus the ~16 hand-written call sites inside
the cluster.

`pretty_deprecated` is wanted regardless — it is going away — so this phase has
value even if the rest stalls.

**Phase B — add the new method.** Add a `pretty` const_method taking
`PpSink &`, returning `void`, to the same IDL. Regenerate. Nothing calls it yet,
so the tree stays green.

**Phase C — implement, subsystem by subsystem**, bottom-up:
printable2 → stringtable2/object2 → procedure2/tokenizer2 → expression2 →
interpreter2 → reader2. This is the real work, ~54 types. See the constraint
below: phase B has to stub all of them first, so this phase replaces stubs
rather than adding methods.

**Phase D — switch call sites** from `pretty_deprecated` to `pretty`.

**Phase E — delete** `pretty_deprecated` from the IDL, regenerate, remove the
old implementations.

### How the facet model constrains this (RC, 2026-08-09)

**The vtable lives with the fat object pointer, not with the object.** At
runtime a valid FOP carries a vtable represented by a specific
`IPrintable_DBar`. `IPrintable_Any` is the placeholder the compiler is induced
to use where the destination type is not yet known — an FOP before it is
initialised. So

```cpp
/* IPrintable_Any.hpp:59 */
[[noreturn]] bool pretty(Copaque, const ppindentinfo &) const override { _fatal(); }
```

means "called through an uninitialised FOP", **not** "this type has not
implemented the method". An earlier reading of this ticket had it as a runtime
fallback and concluded phase B left a runtime landmine. That was wrong, and
wrong in the safe direction: the real constraint is stricter and is enforced by
the compiler.

**Adding a method to `Printable.json5` is not separable.** `APrintable`,
`IPrintable_Any` and `RPrintable` must agree with every `IPrintable_DBar`, so
regenerating the facet side alone does not compile until all the per-type
implementations are regenerated too. Each regenerated `IPrintable_DBar` then
delegates to `DBar`'s method, which must exist.

So phase B cannot be inert, and the sequence needs one more step:

**Phase B — add the method, regenerate everything, and stub every D-type.**
All 54 implementations need *a* `pretty(PpSink&)` for the tree to compile, even
an empty one. That stub pass is mechanical and is what makes the rest
incremental; without it, phases B and C are a single 54-type flag day.

**Phase C then replaces stubs with real implementations**, subsystem by
subsystem, and each is independently verifiable — a stubbed type renders
`STUB:<classname>`, so any output containing `STUB:` names exactly which
printers are still pending.

### The stub cannot bridge to pretty_deprecated — settled 2026-08-09

Asked whether phase B's stub could delegate to the legacy method, so the
intermediate state would render correctly. **It cannot**, and the reason is
structural rather than awkward. `ppindentinfo` carries:

```cpp
/* xo-indentlog/include/xo/indentlog/print/ppindentinfo.hpp */
ppstate *      pps_;    /* legacy xo-indentlog pretty state (flyweight) */
std::uint32_t  ci0_;    /* current indent column */
std::uint32_t  ci1_;    /* ci0 + one indent level */
bool           upto_;   /* true: fit on remainder of line; false: break */
```

Three of the four cannot be supplied from a `PpSink`:

- **`ppstate *`** is xo-indentlog's own state, and a PpSink has none —
  xo-ppsink is deliberately ostream-free and independent of xo-indentlog.
  Manufacturing one reintroduces the stack this migration removes.
- **`ci0_`/`ci1_`** are the caller's indent columns. Under PpSink, indentation
  is the sink's internal business; `FlatSink` has no notion of it at all.
- **`upto_`** *is* the two-pass protocol: legacy calls `pretty()` with
  `upto=true` to ask "does this fit?", then again with `upto=false`. PpSink is
  single-pass and resolves fitting afterwards from the token stream, so no
  value of `upto` carries the intended meaning.

The bridge would have to reconstruct precisely the state the new design exists
to eliminate. (RC's judgement, confirmed against the header.)

**So phase B's stubs cannot bridge** — they render `STUB:<classname>` instead
(see the phase B section). That marker is how phase C is steered: an
unconverted type names itself in the output, which makes "what is still
pending" observable rather than tracked by hand, and unlike silence it cannot
be mistaken for a type that legitimately renders nothing.

**The bridge does invert, though**, and that is worth using. Implementing
`pretty_deprecated(ppindentinfo)` in terms of `pretty(PpSink&)` is at least
plausible — render into a sink, measure the width, answer the fit question —
because it consumes single-pass output rather than manufacturing two-pass
state. Unverified, but if it holds, a converted type's legacy method becomes a
delegate rather than a maintained duplicate, and phases D and E get much
cheaper: no window in which both implementations must be kept correct by hand.

## Phase A0, done 2026-08-09 — and the prerequisite nobody would have guessed

### 35 of the 54 genfacet targets did not exist

`xo_add_genfacetimpl()` creates no target at all unless `share_<FACET_PKG>` is
already defined at the point of the call
(`xo-cmake/cmake/xo_macros/xo_cxx.cmake`):

```cmake
if(NOT TARGET share_${GF_FACET_PKG})
    message(STATUS "xo_add_genfacetimpl: share_${GF_FACET_PKG} not available; skipping ${GF_TARGET}")
    return()
```

In **xo-object2, xo-procedure2, xo-reader2 and xo-stringtable2** nothing had
found `xo_printable2` by then, so those targets were never created and
`cmake --build … -- <target>` failed with a bare `No rule to make target` —
with nothing connecting that to the cause. xo-object2 alone emitted 14 skips,
and they were not only Printable: `share_xo_alloc2` was missing too, so the
`gcobject` facetimpls were equally uncreatable.

**xo-expression2 and xo-interpreter2 worked only by accident.** Their utest
declares `xo_dependency(${UTEST_EXE} xo_numeric)`, which finds `xo_printable2`
transitively. xo-reader2 also has `add_subdirectory(utest)` ahead of its
genfacet blocks, but its utest declares no dependencies — so reordering alone
would NOT have fixed it. The dependency has to be found explicitly:

```cmake
find_package(xo_printable2 CONFIG REQUIRED)
find_package(xo_alloc2     CONFIG REQUIRED)
```

added before the genfacet blocks in each of the four. Skips then go to zero,
including the self-referential `FACET_PKG xo_procedure2` / `xo_reader2` entries
— those resolve through the transitive closure, contrary to what this ticket's
author first assumed and wrote into the comments before measuring.

Verify with: reconfigure and count `not available; skipping`. Expect **zero**.
That count is the only signal; nothing fails.

### Three hand-edits to generated files, relocated to their sources

Regenerating reverts anything hand-applied to generated output. A0 found three:

| hand-edit | proper home |
|---|---|
| include ordering (xo topological policy) | post-generation `xo-clang-format-includes --fix` |
| `#include <xo/alloc2/Allocator.hpp>` in `Printable.hpp`, added by `b1add3bb` | the IDL's `user_hpp_includes` |
| `_drop` / `_has_null_vptr` **absent** | nothing to relocate — the templates had moved on |

The third is a semantic change riding along: `_drop` is a new pure virtual on
`APrintable`, implemented in `IPrintable_Xfer` as `_dcast(d).~DRepr()`, so
facet-level destruction went live across all 54 impls as part of A0. That is
precisely why A0 is a separate commit — if destruction misbehaves it is
attributable here, not tangled with a rename.

Include ordering cannot be fixed in the IDL, incidentally: the template emits
the IDL's includes *before* its own fixed `xo/facet/*` block, and the correct
position of `xo/indentlog/…` relative to `xo/facet/…` is a topology fact
(`xo-gen-clang-format` derives it from `subsystem-edges`; facet is priority
141, indentlog 152). The generator should not know include policy; the
formatter should.

### The acceptance test for A0

```bash
git diff > before.diff
# regenerate printable2 + all 54, then:
xo-clang-format-includes --fix --style <generated-style> $(git diff --name-only | grep -E '\.(hpp|cpp)$')
git diff > after.diff
diff -q before.diff after.diff     # must be identical
```

**Regeneration must be a no-op.** It was not before A0, and is now. Until that
holds, no later phase's diff can be trusted to contain only its own change.

Result: 61 subsystems build clean; sweep 34 ok / 26 no-tests / 1 failed
(`xo-jit`, the documented baseline) — unchanged by A0.

## Phase A, done 2026-08-09 — and the order that makes it work

223 files across 7 subsystems; `pretty_deprecated` in 221 of them. Build clean,
sweep unchanged (34 ok / 26 no-tests / 1 `xo-jit`), regeneration idempotent, no
`pretty(ppindentinfo)` left tree-wide.

### The cycle, in this order. Two of the three steps fail silently if reordered

```bash
# 1. edit xo-printable2/idl/Printable.json5
# 2. INSTALL printable2 -- not optional, see below
xo-build -q --build --install xo-printable2
# 3. regenerate the facet, then all 54 impls
cmake --build xo-printable2/.build -- xo-printable2-facet-printable
#    ... then each xo-<sub>-facetimpl-printable-<type> target
# 4. hand-edit the DFoo implementations
# 5. canonicalize includes LAST
xo-clang-format-includes --fix --style <style> $(git diff --name-only | grep -E '\.(hpp|cpp)$')
```

**Step 2 is the trap.** `xo_add_genfacetimpl` resolves the facet IDL from the
*installed* share dir (`get_target_property(share_<pkg> path)`), not the source
tree. Edit the IDL, skip the install, and all 54 impl targets run, report
success, and regenerate against the OLD interface — producing no diff at all.
Nothing warns. That cost a full regeneration cycle here before the missing
`pretty_deprecated` in an impl gave it away.

**Step 5 must be last.** Canonicalizing before the hand-edits leaves the
hand-edited files out of policy order, and the next cycle's formatter pass then
changes them — which looks exactly like the generator being non-idempotent. It
is not; the acceptance test just has to run after the formatter has settled.

### A latent bug the rename exposed

xo-interpreter2 had **two** copies of `IPrintable_DLocalEnv.hpp`. Its IDL says
`output_impl_subdir: "env"`, so regeneration maintains `env/`; but
`LocalEnv.hpp` still included `detail/IPrintable_DLocalEnv.hpp`, an orphan from
when the IDL said `detail`, committed to git and regenerated by nothing. Its
sibling `IPrintable_DClosure.hpp` genuinely does live in `detail/`, which is
why the stray copy looked unremarkable.

Invisible while both declared `pretty`. The rename updated one and left the
other, and the compiler found it at once.

**The detector generalises:** every impl header had to change, so any that
did *not* is an orphan.

```bash
for f in $(git ls-files | grep -E 'IPrintable_D.*\.(hpp|cpp)$'); do
    git diff --quiet "$f" && echo "ORPHAN $f"
done
```

Exactly one tree-wide. Fixed by pointing `LocalEnv.hpp` at `env/` — which its
own `.cpp` already used — and deleting the orphan. Worth re-running that check
at each later phase, since it costs nothing and catches divergence that
otherwise waits for a rename to expose it.

### Renaming precisely

`pretty` is not one identifier here. `pps->pretty(value)` is **ppstate's** own
method from xo-indentlog's printer API, nothing to do with the facet, and a
blanket `pretty(` substitution breaks it. The two are cleanly separable by
argument — no `pps->pretty(ppii)` exists — so the rename keys on the
ppindentinfo argument:

```bash
sed -e 's/\bpretty(const ppindentinfo/pretty_deprecated(const ppindentinfo/g' \
    -e 's/\bpretty(const xo::print::ppindentinfo/pretty_deprecated(const xo::print::ppindentinfo/g' \
    -e 's/\([.>]\)pretty(ppii)/\1pretty_deprecated(ppii)/g'
```

`pretty_struct` and `print_pretty` are untouched by `\bpretty\b` anyway
(underscore is a word character); `pretty.hpp` in includes is why the pattern
requires the open paren.

## Phase B, done 2026-08-09 — and the phase C work-list

168 files, +1021/-1: a second `const_method` on the facet, its regenerated
consequences across the 54 impls, and 55 stubs. Build clean, sweep unchanged
(34 ok / 26 no-tests / 1 `xo-jit`), regeneration idempotent. Nothing calls the
new method yet, so no rendered output changed.

The facet now carries both, and `IPrintable_Any` terminates both:

```cpp
virtual bool pretty_deprecated(Copaque data, const ppindentinfo & ppii) const = 0;
virtual void pretty(Copaque data, PpSink & sink)                        const = 0;
```

The IDL needed a `PpSink` entry in `types:` (`xo::pp::PpSink`) and
`<xo/ppsink/PpSink.hpp>` in `includes:`. **No new dependency** — xo-printable2
already declares xo-ppsink (`xo-deps --deps-of=xo-printable2`), reaching it via
xo-facet → xo-arena.

### The stub

Inline in each D-type header, immediately after `pretty_deprecated`, so there
is no `.cpp` edit and nothing to satisfy at link time:

```cpp
/* PHASE B STUB -- not yet converted by phase C.  Renders a marker rather
 * than nothing, so an unconverted printer is VISIBLE in output instead of
 * silently absent.
 * See .xo-backlog/xo-printable2/issues/01-aprintable-pretty-ppsink.md
 */
void pretty(xo::pp::PpSink & sink) const { sink.put("STUB:DFoo"); }
```

**Revised 2026-08-09 (RC): the stub emits `STUB:<classname>`, not nothing.**
It first rendered nothing, and this ticket said so — wrongly, and in a way that
mattered, because the phase C steering advice depended on it.

The marker gives a **runtime** detector alongside the two static ones. Silence
is indistinguishable from "this type legitimately renders as nothing"; a marker
is not. Once phase D switches call sites, an unconverted printer announces
itself in rendered text.

It also strengthens the case for closing this ticket. "Did we visit all 55?"
previously rested on a grep over source; it now also rests on **no rendering
anywhere containing `STUB:`**, checkable against real output including the
REPL's.

Mechanical notes for anyone regenerating or extending the stubs:

- **The class name is not the filename stem.** `DPrimitive.hpp` declares
  `class Primitive`; `TypeRef`, `ParserStack` and `ParserResult` are not
  `D`-prefixed at all. Derive it by scanning back to the nearest enclosing
  `class`/`struct` at *shallower indentation* than the stub, then check every
  site resolved and every name is distinct — 53 and 53 when this was done.
- **`put`, not `pp`.** `sink.put(...)` writes raw, as `pretty_array.hpp` does
  for its brackets. `pp` would route a string literal through the Prettifier
  machinery for nothing.

Deliberately **not** `noexcept`, even where the neighbouring
`pretty_deprecated` is: phase C's real implementations might throw, and the
facet's own declaration is not noexcept either.

Deliberately **not** `noexcept`, even where the neighbouring
`pretty_deprecated` is: a do-nothing stub cannot throw, but phase C's real
implementations might, and the facet's own declaration is not noexcept either.

`xo::pp::PpSink` resolves in these headers with no added include — confirmed by
converting one header first and building, before touching the other 54.

### The work-list IS a query — do not copy it into this ticket

55 sites, one per D-type header, no header carrying more than one:

```bash
grep -rl 'PHASE B STUB' --include=*.hpp xo-*/ | grep -v '/\.build/'   # 55 files
```

| subsystem | stubs |
|---|---|
| xo-reader2 | 24 |
| xo-expression2 | 12 |
| xo-object2 | 8 |
| xo-interpreter2 | 8 |
| xo-stringtable2 | 2 |
| xo-procedure2 | 1 |

(counts as of 2026-08-09; re-run the grep rather than trusting them)

**Phase C progress is therefore derived, not tracked.** Each conversion deletes
its stub marker, so the count falls monotonically and reaching zero *is* phase C
being complete. Same principle as the `Milestone:` lines: a hand-kept checklist
would drift the first time someone converted a type and forgot to tick it.

### The `const noexcept` variant, and a check that confirmed the wrong set

Four D-types declare the legacy method as `const noexcept;` rather than
`const;`:

```
xo-interpreter2/include/xo/interpreter2/DLocalEnv.hpp
xo-interpreter2/include/xo/interpreter2/DVsmDefContFrame.hpp
xo-interpreter2/include/xo/interpreter2/DVsmIfElseContFrame.hpp
xo-interpreter2/include/xo/interpreter2/DVsmSeqContFrame.hpp
```

The first stub pass used a pattern requiring `const;` and so silently covered
51 of 55. **The verification missed it too**: "51 declarations, 51 stubs, every
declaration matched" — because both numbers came from the same faulty pattern.
A self-consistent count proves nothing when both sides derive from one query.
The build caught it, as it caught the orphan in phase A.

Any later pass over these declarations must accept both forms. Recovery was
free only because the insertion script was idempotent — re-running it on the
four inserted exactly four and left the other 51 untouched.

## Phase C: the per-printer cycle (established 2026-08-09)

RC's workflow: **introduce printers one at a time, bottom-up, and stop to
review after each.** Rendering differences need reviewing individually, so
roughly 55 commits. The harness's job is therefore to *surface* differences for
judgement, not to force equality — where old and new disagree, that is a
decision, not a test failure to be silenced.

### The cycle

```cpp
render_deprecated(x, margin)  // toppstr2(ppconfig{margin}, x)
                              //   -> ppdetail<obj<APrintable,D>> -> pretty_deprecated
render_pretty(x, margin)      // toppstr(PpConfig::scratch(margin), x)
                              //   -> Prettifier<obj<APrintable,D>> -> pretty

REQUIRE(modern == legacy);         // differential -- SCAFFOLDING, dies at phase E
REQUIRE(modern == tc.expected_);   // absolute    -- the coverage that SURVIVES
```

**Margin is the case variable**, in the schedule style
(`Testcase_Render`/`s_testcase_v`/indexed loop with `INFO`): it is what makes
the two protocols disagree if they are going to.

`PpConfig::scratch(margin)` arrived with `522799d7` (RC, 2026-08-09) and
replaced the hand-built
`PpConfig().with_soft_right_margin(margin).with_style(PpStyle::plain())` in all
four render tests. It carries the 64k logbuf, a unique arena name and
`PpStyle::plain()`, so a `render_pretty` helper no longer has to remember the
style argument to keep its expectations free of ANSI escapes — see the colour
sections below for why that argument was there.

Worked example: `xo-stringtable2/utest/printable_render.test.cpp`.

### BOTH assertions are required, and the second is the point

A purely differential test is scaffolding. It cannot survive phase E — deleting
`pretty_deprecated` removes the thing it compares against — so a cycle that
only ever asserts `modern == legacy` ends with 55 tests to delete and **no
coverage gained**.

This ticket's author wrote exactly that on the first attempt, having already
argued the opposite, and it was caught only because RC asked. Every printer's
test must pin its **observed** output as a literal, so that when the legacy half
goes away the expectation remains.

There is currently **no test anywhere that renders any of these 55 types**, so
phase C is first-time coverage, not overhead on the way to a refactor. That is
the strongest reason to do it per-printer rather than in bulk.

### Infrastructure this needed

- **`xo::pp::toppstr`** — `.xo-backlog/xo-indentlog2/issues/01`. The
  new-protocol render. Did not exist; 23 files hand-rolled it.
- **`Prettifier<obj<APrintable, DRepr>>`** —
  `xo-printable2/include/xo/printable2/detail/pretty_Printable.hpp`, wired in
  via `user_hpp_includes`. The exact ppsink mirror of the existing
  `ppdetail_Printable.hpp`. Without it `toppstr(cfg, x)` falls through
  Prettifier's empty primary template to `operator<<`, and the differential test
  compares two wrong things while passing. Safe to add when it was added
  because nothing rendered a printable obj through a ppsink yet — check that
  again if it is ever reintroduced elsewhere.

### Mutation-check each printer

Green proves nothing here until the comparison is shown to discriminate: both
sides could be falling through to the same fallback. Break the new `pretty()`,
confirm the test fails, revert. Doing this for `DString` also demonstrated
`DUniqueString`'s delegation chain was live rather than accidentally passing.

### Done so far

Which types remain is the `Progress:` query, not this table; the table is for
what each conversion *taught*, which a count cannot carry.

| type | notes |
|---|---|
| `DString` | `sink.pp(&chars_[0])` — `ppdetail_atomic` is a bare `pps()->write(x)`, a leaf with no quoting or framing |
| `DUniqueString` | pure delegation, as the deprecated form was |
| `DInteger`, `DBoolean` | leaves |
| `DFloat` | leaf, but the two stacks format doubles by different code (`ppstate::write` vs `Prettifier<double>`); six values pinned, not one |
| `DList` | deliberate divergence: legacy printed `(...)`, ppsink prints the elements structurally. Pinned in `xo-gc/utest/Object2.test.cpp` |
| `DRuntimeError` | first `pretty_struct`; the divergence below |
| `DArray` | first variable-arity sequence; `begin(1)` to align elements, see below |
| `DDictionary` | keyed sequence; `DArray`'s framing plus a per-entry group so the value folds after the key, see below |
| `Primitive<Fn>` | first struct whose field VALUES leave the line; exposed the field-value indent divergence and a second colour gate, see below |
| `TypeRef` | first NON-D-type printer, so the first needing its own `Prettifier<>`; and the first field whose legacy `cond()` has no ppsink equivalent, see below |
| `DVariable` | first printer NESTING an already-converted one (`TypeRef`), so the first where the field-value indent divergence compounds per level, see below |
| `DVarRef` | first field taking ppsink's leaf FALLBACK (`Binding` has only an `operator<<`) — the silent case; and a legacy null deref, see below |

The four leaves are **degenerate**: an atomic leaf has no break points, so they
render identically at every margin, even margin 4 against 17 characters of
content. They establish the cycle; they do not stress it.

### The first structured printer: what `DRuntimeError` settled (2026-08-09)

`DRuntimeError` is `pretty_struct` with two `DString` fields — two levels, the
smallest thing that can disagree about break placement. Rendered through both
protocols at margins 80 / 40 / 16, output observed rather than predicted
(`xo-object2/utest/printable_render.test.cpp`, `s_error_v`):

- **margin 80** — one line, identical.
- **margin 40** — struct breaks, each field on its own line at indent 2.
  Identical. So the two protocols agree on *where the struct breaks* and on the
  struct-level indent.
- **margin 16** — each field breaks from its own value, and the continuation
  indent differs: **legacy 4, ppsink 3**.

The margin-16 difference is real and deliberate. Legacy adds another
`indent_width_` (2) below the field; ppsink adds `xo::pp::tag_config::value_offset`,
which is 1 (`xo-ppsink/include/xo/ppsink/tag.hpp:46`), so a broken value hangs
one column past its `:name` instead of lining up with the next nesting level.
Reviewed and kept: it is a named, configurable constant in ppsink where legacy's
was neither. Pinned in the test.

**A trap the leaf tests hid — `ppconfig::ugly()`.** The template test used
`xo::print::ppconfig::ugly()` for the legacy render, and every test copied from
it inherited that. `ugly()` sets `indent_width_ = 0`
(`xo-indentlog/include/xo/indentlog/print/ppconfig.hpp:29`), so under it a broken
struct's fields land in column 0 — the comparison silently stops being about
indent at all. Harmless for a leaf, which never breaks; wrong the moment a
printer has structure. **Use a default-constructed `ppconfig` with
`right_margin_` set** (`indent_width_ = 2`, matching `xo::pp::PpConfig`).
`xo-object2`'s test now does; `xo-stringtable2`'s still says `ugly()` and should
be changed if anything structured is ever added there.

### The first sequence printer: what `DArray` settled (2026-08-09)

`DArray` is the first printer whose arity is not fixed, so the margin decides
both *whether* it breaks and *how many* elements share a line. Its ppsink form
is `sink.put("[").begin(1)` / `split(1)` between elements /
`sink.end().put("]")` — structurally `DList`, which landed earlier, with `[` `]`
for `(` `)`, plus the `begin` offset below.

Rendered through both protocols and observed
(`xo-object2/utest/printable_render.test.cpp`, `s_array_v`):

- **fits** (`[]`, `[1]`, `[1 2 3]` at margin 80) — identical.
- **breaks** (`{100,200,300}` at margin 8, and again at 4) — both stacks align
  elements 2..n under element 0, and reach that alignment differently:

```
legacy      ppsink
[ 100       [100
  200        200
  300]       300]
```

**Legacy pads; ppsink offsets.** Legacy writes a space after the bracket
(`pps->indent(std::max(pps->indent_width(), 1u) - 1)`, "indent, but credit
initial `[`") so element 0 starts *at* the continuation indent of 2. ppsink's
`begin(1)` credits the bracket in the indent itself, so element 0 follows `[`
immediately and the continuation indent is 1 to match. Same alignment, one
column narrower, and no padding whitespace in output that is otherwise
whitespace-free.

RC's call, 2026-08-09. **It makes `DArray` deliberately unlike
`Prettifier<vector<T>>`**, which indents a broken sequence by a full nesting
level — `"[1,\n  2,\n  3]"`, pinned in
`xo-indentlog2/utest/PrettyVector.test.cpp`. Worth stating plainly because the
first attempt here went the other way *on* that consistency argument: plain
`begin()` reproduces the vector convention and drops the alignment legacy had.
Alignment won. Anything reconciling the two later should change the vector
prettifier, not `DArray`.

Note `split(0)` is not the knob for this. `split(spaces, offset)` uses `spaces`
only when the group *fits*, so `split(0)` would render `[123]` on one line while
leaving the break indent untouched. `split(1, -1)` would also align, but sets
the offset once per separator instead of once per group.

Margin 4 renders the same as margin 8 — an element wider than the margin has no
further recourse — which is worth pinning precisely because it is the case where
a line-breaker might instead degenerate.

**A flat sequence cannot check the thing most likely to be wrong.** Every case
above passes whether the offset composes with the enclosing indent or replaces
it, because there is only one level. So `DArray-render-nested` renders
`[[100 200] [300]]` at margins 80 / 12 / 6: at 12 the outer breaks and the inner
arrays stay flat, aligned at column 1; at 6 both break and the inner elements
land at column 2, under an inner `[` that is itself at column 1. That last
number is the one that distinguishes composing from resetting.

Mutation-checked three ways, since green proves nothing until the comparison is
shown to discriminate: widening `split(1)` to `split(2)`, deleting the
`begin()`/`end()` pair, and dropping the `begin` offset back to `begin()` each
fail both test cases.

### `DDictionary`: the entry is a group, and the key's width stops mattering (2026-08-09)

`DDictionary` keeps the `DArray` framing — `put("{").begin(1)`, `split(1)`
between entries, `end().put("}")` — and adds one thing `DArray` had no need
for: **each entry is its own group**, with a fold point after the colon.

```cpp
sink.begin(1);
sink.pp(key);
sink.put(":");
sink.split(1);          // fits -> "k: v";  breaks -> value on its own line
sink.pp(value);
sink.put(";");
sink.end();
```

The `;` **terminates** rather than separates, so the last entry carries one too;
that is legacy's shape and it was kept.

The first attempt wrote `put(": ")` and no entry group. RC rejected it: *"if
`{a: 1; b: 2;}` doesn't fit on one line, then the tag `k:` should be folding …
after all, instead of being `k`, the attribute name could have been
`keep_going_until_you_get_very_close_to_the_right_margin`."* Exactly so — with
the value starting inline, its every subsequent line is positioned by an
accident of how long the key was. Two consequences worth stating separately:

- The **group** is what makes the fold conditional. A bare `split` with no
  `begin`/`end` around the entry inherits the *dictionary's* fit decision, so
  every entry would break the moment the dictionary did.
- `split(1)` rather than `put(": ")` costs nothing when the entry fits — it
  emits the same single space — so no fitting rendering changed.

Observed (`s_dict_v`, `DDictionary-render-nested`), never predicted:

| | legacy | ppsink |
|---|---|---|
| empty | `{ }` | `{}` |
| fits | `{ a: 1; bb: 22; }` | `{a: 1; bb: 22;}` |
| dict breaks, entries fit | `{ a: 1;`<br>`  bb: 22; }` | `{a: 1;`<br>` bb: 22;}` |
| entry breaks too (margin 4) | *unchanged* | `{a:`<br>`  1;`<br>` bb:`<br>`  22;}` |

The first three rows are the padding-vs-offset divergence `DArray` already
settled. The fourth is new, and is the first place a **ppsink printer has
recourse legacy does not**: legacy has no break point after a key, so it stops
degrading while ppsink keeps going. Same story with a wide key at margin 30 —
legacy emits a 41-column line; ppsink folds.

Nested, `{k: {a: 1; b: 2;}; n: 3;}`:

margin 16, ppsink — the entry folds, and the inner dictionary then fits:

```
{k:
  {a: 1; b: 2;};
 n: 3;}
```

margin 10, ppsink — the inner dictionary breaks as well:

```
{k:
  {a: 1;
   b: 2;};
 n: 3;}
```

legacy, at **both** margins — identical, because it cannot fold after `k:` and
so has nothing left to give:

```
{ k: { a: 1;
       b: 2; };
  n: 3; }
```

Legacy is *identical* at 16 and 10 — no recourse — while ppsink folds the entry
at 16 and then breaks the inner dictionary at 10. The inner dictionary opens at
the running indent (2), so its own `begin(1)` puts its entries at 3: the offset
**composes** with the enclosing indent rather than resetting. That is the
assertion a flat dictionary cannot make, and the reason for a nested test here
as for `DArray`.

Note what folding also bought: because the value always starts at
`entry-indent + 1`, the question of aligning a continuation to a *column* never
arises for a dictionary. `PpSink` has no column-relative group — `begin(offset)`
is defined against the running indent (`PpSink.hpp:166-170`), `lpos()` reports a
column but the running indent is not exposed, and `FlatSink::lpos()` is
`nullopt` — and with the fold in place it does not need one.

**Out of scope, RC's call:** the more readable style is the one that lets nested
content flow on past the key and wrap liberally, rather than folding as soon as
it will not fit. That needs three layout choices per group (fit / flow / break)
where the sink has two. Not now.

Mutation-checked five ways: outer `split(1)`→`split(2)`, outer
`begin(1)`→`begin()`, dropping the outer `begin`/`end`, dropping the **entry**
`begin`/`end`, and replacing `put(":")`+`split(1)` with `put(": ")` — each fails
both test cases.

`DStruct` (`xo-object2/include/xo/object2/DStruct.hpp:144`) still carries a stub
and always will: it is a dead skeleton. Nothing in the tree references it, it
has no `.cpp` (no member is defined anywhere, `pretty_deprecated` included), it
is in no `CMakeLists.txt`, it has no facet impls and no `Printable.json5` entry,
and the header does not even compile — it still includes the v1 `xo/gc/
Collector.hpp`, which is not on xo-object2's include path. Phase B stubbed it by
pattern-matching `pretty_deprecated`. Skipped, with RC's agreement; the stub
count therefore has a floor of 1 in xo-object2.

### `Primitive<Fn>`: the first struct whose field values leave the line (2026-08-09)

`xo-procedure2`'s only printer, and the last one below expression2 / interpreter2
/ reader2. It is a class **template** (`Primitive<Fn>`, aliased as
`DPrimitive_gco_0`, `..._1_gco`, `..._2_gco_gco`, `..._2_dict_string`,
`..._3_dict_string_gco`), so the definition goes out of line in the header
beside `pretty_deprecated`, not in `DPrimitive.cpp`.

The body is a direct transcription of the deprecated one:

```cpp
sink.pretty_struct("Primitive<Fn>",
                   xo::pp::field("name", name_),
                   xo::pp::field("td",   fn_td_),
                   xo::pp::field("fn",   fn_));
```

Two new includes in `DPrimitive.hpp`: `<xo/ppsink/pretty_struct.hpp>` and
`<xo/reflect/TypeDescr_pp.hpp>` (the `Prettifier<TypeDescr>` for the `:td`
field). `xo-procedure2` declares neither `xo_ppsink` nor `xo_reflect` in
`src/procedure2/CMakeLists.txt`; both already arrived transitively, as
`<xo/reflect/Reflect.hpp>` and `<xo/indentlog/print/pretty.hpp>` did before this
change. Left alone rather than fixed here, so this stays one printer per commit.

**Why this one is not just another `pretty_struct`.** `DRuntimeError`'s fields
are short `DString`s: they always fit, so nothing about a *broken* field was ever
pinned. `Primitive`'s `:td` renders a `TypeDescr` whose `:canonical_name` alone
is 78 columns. Two things fall out, both first sightings:

**1. A broken field's value indents differently — legacy 4, ppsink 3.**

```
legacy                     ppsink
<Primitive<Fn>             <Primitive<Fn>
  :name cwd                  :name cwd
  :td                        :td
    <TypeDescr ...            <TypeDescr
  :fn 1>                      ...
```

Both put `:td` in column 2. Legacy then puts its value at 2 + `indent_width`
(2) = 4; ppsink at 2 + `PpStyle::tag_value_offset` (1) = 3
(`Prettifier<field_impl>`, `pretty_struct.hpp:101`). This is not specific to
`Primitive` — it applies to **every** struct field value that goes to its own
line, so expect it in expression2 / interpreter2 / reader2 too.

**2. Legacy's `:td` cannot break at any margin; ppsink's folds.** Not because
the two-pass protocol is weaker here, but because legacy's `TypeDescr` rendering
is *already ppsink underneath and already flat*:
`TypeDescrBase::display(std::ostream&)` streams `xo::pp::xtag` through a
`FlatSink` (`TypeDescr.cpp:337`, `tag_ostream.hpp`), reached via
`operator<<(ostream&, const TypeDescrBase&)`. Legacy therefore renders a
143-column line at margin 20 and is byte-identical at margins 80 and 20. Same
shape as `DDictionary`'s long-key case, different cause.

**A second colour gate.** `render_deprecated` in the other
`printable_render.test.cpp` files clears only
`xo::tag_config::tag_color_enabled`. That is not enough here: the nested
`TypeDescr` reaches ppsink (above), which reads colour from `PpStyle`, so the
legacy rendering came back with ANSI escapes on the *inner* field names and none
on the outer ones. Fixed with `xo::pp::default_style_guard
plain(xo::pp::PpStyle::plain())` — the tool `PpStyle.hpp` documents for exactly
this, reaching the `FlatSink`s that convenience entry points build internally.
Any legacy-side expectation that transitively renders something already migrated
needs both gates.

**`:fn 1`, in both renderings.** A function pointer has no `operator<<`, so it
converts to `bool`. Legacy printed `1`; ppsink prints `1`. Not a divergence and
not introduced here — recorded so it is not mistaken for ppsink damage. Making
it useful (a symbol name, or dropping the field) is an output-visible change and
belongs in its own commit.

**`:id` is scrubbed in the test.** `TypeId` is a process-wide counter handed out
in reflection order, so the value inside `:td` depends on how many other types
this binary reflects first — an unrelated new test file would move it.
`scrub_type_id()` replaces the digits with `N`; every other column stays pinned
exactly. It is xo-reflect's field, pinned by xo-reflect's own tests.

Pinned in the new `xo-procedure2/utest/printable_render.test.cpp` at margins
200 / 80 / 20, over `ObjectPrimitives::make_cwd_pm`. Mutation-checked four ways:
struct name `"Primitive<Fn>"`→`"Primitive"`, dropping the `:fn` field, field name
`"td"`→`"tdx"`, and wrapping the whole struct in an extra `begin()`/`end()` (the
layout mutation) — each fails.

### `TypeRef`: a printer with no facet, and a field legacy `cond()` cannot hand over (2026-08-09)

The bottom of xo-expression2 — two fields, depending on nothing else in the
subsystem — and the first conversion where the *dispatch* had to be built as
well as the printer.

**`TypeRef` is not a D-type.** It has no facet impl, so no regenerated
`IPrintable_DTypeRef` calls its `pretty()`. Legacy reached it through a
hand-written `print::ppdetail<TypeRef>` in `TypeRef.hpp`; ppsink needs the exact
mirror, added beside it:

```cpp
template <>
struct Prettifier<xo::scm::TypeRef> {
    static void print(PpSink & sink, const xo::scm::TypeRef & x) { x.pretty(sink); }
};
```

Without it the empty primary template sends a `TypeRef` to `operator<<`, which
it does not have — a compile error here, but the same omission on a type that
*does* have an inserter would compile and silently render flat. The ticket's
`Prettifier<obj<APrintable,DRepr>>` note makes this point for the facet types;
it applies to the three non-`D` exceptions too (`TypeRef`, `ParserStack`,
`ParserResult`).

**`cond(td_, td_, "null")` has no ppsink counterpart, and the substitute is a
branch in the printer.** Legacy rendered `:td null` for an unresolved TypeRef.
ppsink has no `cond`, and the two things that look like they would serve do not:

- `Prettifier<TypeDescr>` prints **nothing** for a null descriptor
  (`TypeDescr_pp.hpp`, deliberate — printing `<null>` is an output-visible
  change to xo-reflect and is reserved for its own commit), so
  `field("td", td_)` yields a bare `:td` with no value.
- `field()`'s `present` flag omits the field *entirely*, which is a different
  statement: "this TypeRef has no td field" rather than "its td is null".

So `TypeRef::pretty()` uses `struct_open()` and branches, supplying the word
itself. Output matches legacy exactly. Recorded because an unresolved TypeRef is
the normal pre-typecheck state, not an edge case, and because the same shape
will recur wherever a legacy printer used `cond()` — `DDefineExpr` and
`DProgressSsm` are the other two.

RC, on review: **if/when ppsink supports `cond()`, this printer is a use case for
it.** `.xo-backlog/xo-ppsink/issues/02` is the facility inventory and now carries
the two shape constraints this call site imposes — arms of differing type, and
usable as a field *value* rather than a sink-writing function.

**`quot`, not `unq`.** First quoting in the migration. Legacy used
`xo::print::quot`, which always quoted; `xo::pp::quot` is its exact counterpart
and `xo::pp::unq` (quote only when bare would be ambiguous) is not — it would
render `t:1` bare and turn an empty id into nothing at all rather than `""`.
Both forms are pinned, `"t:1"` and `""`, so the choice cannot be reverted
silently. The nicer-looking `unq` remains available if the rendering is ever
revisited deliberately.

Everything else was already settled by `Primitive<Fn>`, and reappears for the
same reason — `:td` is a `TypeDescr`: the broken-field-value column (legacy 4,
ppsink 3), legacy's `:td` unable to break at any margin because its path is
already a FlatSink, and the need for **both** colour gates in
`render_deprecated`.

Pinned in the new `xo-expression2/utest/printable_render.test.cpp` over three
TypeRef states (id only / td only / both) at margins 200 / 80 / 40 / 20.
The type-variable name is supplied rather than generated:
`TypeRef::generate_unique()` draws on a process-wide counter, so a generated
name would move whenever an unrelated test made a TypeRef first — same hazard as
the nested `TypeDescr`'s `:id`, which is still scrubbed.

Mutation-checked four ways — struct name `"TypeRef"`→`"TypeRefX"`, `quot`→`unq`,
dropping the null branch, and an extra `begin()`/`end()` around the struct —
each fails exactly one test case.

`xo-expression2/utest` gained `xo_testutil` and `xo_indentlog2`, and
`pkgs/xo-expression2.nix` the matching check-only `xo-testutil`
(same fix as `9526eeb8` for xo-object2 and its repeat for xo-procedure2 — a
utest dependency added in CMake is a nix package edit every time).
`cli11` was added alongside it and **removed again by RC**: xo-procedure2's
utest main parses arguments, `xo-expression2/utest/expression2_utest_main.cpp`
does not. Copying a nix package edit from a sibling copies its dependencies too;
check each one has a consumer here.

### `DVariable`: the indent divergence compounds per level (2026-08-10)

The first converted printer that **nests another converted printer** — its
`:typeref` field is a `TypeRef`, pinned the day before. Nothing new had to be
built: `xo::pp::field("typeref", typeref_)` finds `Prettifier<TypeRef>`
(`TypeRef.hpp`) and the nested printer composes. That is the first evidence the
bottom-up order is paying, rather than merely being tidy.

Body is a plain `pretty_struct` — no `struct_open` branch, unlike `TypeRef`,
because the one conditional (`name_` may be null) picks a *value* rather than
deciding whether a field exists:

```cpp
auto name = (name_ ? std::string_view(*name_) : std::string_view(""));
const auto qname = xo::pp::quot(name);

sink.pretty_struct("DVariable",
                   xo::pp::field("name", qname),
                   xo::pp::field("typeref", typeref_));
```

A null `name_` renders `""` — **not** nothing, and not `"null"`. Both stacks
reach it through the same branch, which legacy already had, so this is not a
`cond()` case: the choice happens before either printing protocol sees it.

**What is new: the field-value indent divergence compounds per nesting level.**
`Primitive<Fn>` and `TypeRef` each showed it once — a broken field's value lands
at `indent + indent_width` (legacy, 2) versus `indent + tag_value_offset`
(ppsink, 1). Two levels deep it accumulates, so the nested `<TypeRef` opens at
column 4 vs 3, and *its* fields at 6 vs 4. Observed at margin 80:

```
legacy                             ppsink
<DVariable                         <DVariable
  :name "myvar"                      :name "myvar"
  :typeref                           :typeref
    <TypeRef                          <TypeRef
      :id ""                           :id ""
      :td <TypeDescr ...>>>            :td <TypeDescr ...>>>
```

Same tokens, same break points, different column arithmetic — and the gap widens
with depth rather than staying constant. Worth stating plainly now, because the
deeper printers still to come (`DLambdaExpr`, `DApplyExpr`, and most of reader2)
will diverge by more than this and it will look like a new problem each time.
It is one difference, already reviewed, applied per level.

**Do not "fix" it by retuning a default.** Asked and settled 2026-08-10:
`tag_value_offset = 2` would make the two stacks agree at every level, and it is
a one-line change — but the defaults stay put for the rest of phase C, and the
answer is runtime configuration instead. Reasoning, and the finding that
`indent_width` and `tag_value_offset` live in different config objects at
different levels, are in
`.xo-backlog/xo-ppsink/issues/07-nested-formatting-context.md`.

Pinned in `xo-expression2/utest/printable_render.test.cpp` (`s_dvariable_v`),
seven cases over name present/absent × typeref resolved/unresolved × margins
200 / 80 / 40 / 20. Rendered through `with_facet<APrintable>::mkobj(var)` rather
than the raw pointer — DVariable *is* a facet D-type, unlike `TypeRef`, and that
is the path phase D changes.

Mutation-checked four ways — struct name `"DVariable"`→`"DVariableX"`, dropping
`quot()`, rendering the null name as `"null"`, and swapping the two fields —
each fails exactly one test case.

The fixture needed a real collector (`DX1Collector` + `CollectorTypeRegistry`)
and `InitSubsys<S_expression2_tag>`, because a `DVariable` cannot exist without
an allocator and its `APrintable` facet is registered by `SetupExpression2`.
That is the shape every remaining expression2/interpreter2/reader2 printer will
need, and it is why `TypeRef` was worth doing first: it needed none of it.

### `DVarRef`: the fallback fires silently, and legacy dereferenced null (2026-08-10)

Two fields, `:name` (a string) and `:path` (a `Binding`), and the second is why
this one was worth doing before the interior nodes.

**`Binding` has no `Prettifier<>` and no `ppdetail<>` — only an `operator<<`**
(`xo-expression2/include/xo/expression2/Binding.hpp:58`). So it takes ppsink's
leaf fallback: the empty primary template means not-string-like falls through to
`operator<<` (`Prettifier.hpp`, and the rule is documented on the primary
template itself). Both stacks render `{path:0:3}`, byte-identical at every
margin tested.

That agreement is the point. `TypeRef` was the same situation and **failed to
compile**, because it had no `operator<<` — which made a missing `Prettifier<>`
loud. `Binding` shows the other half: a type that *does* have an `operator<<`
compiles either way and renders flat, with no diagnostic. Pinning it is the only
way to know the fallback fired rather than something else. Directly relevant to
`ParserStack` and `ParserResult` in reader2, which are in the same position.

RC's call on reviewing it: **`Binding` should get a `Prettifier<>` eventually.**
Written up as a follow-up, with the four other cluster types that take the same
branch, in `.xo-backlog/xo-expression2/issues/01-binding-prettifier.md`. Not a
conversion turn — it changes no output, and `s_dvarref_v` already pins the bytes
it must preserve.

**Legacy dereferences a null pointer here.** `pretty_deprecated` does
`std::string_view(*(this->name()))`, and `name()` forwards to
`vardef_->name()`, which has no non-null invariant — `DVariable::pretty` guards
for exactly this. `DVarRef::pretty()` guards too, matching the sibling printer.
That is a change, so it is stated rather than slipped in:

- output-identical wherever legacy is *defined*, i.e. every case in the table
- the null case is pinned in a **separate test** (`DVarRef-anon-render`) that
  renders through `pretty()` only, because there is no legacy rendering to
  compare against — legacy is undefined, not merely different
- mutating the guard's empty string to `"?"` fails that test and only that test

**A legacy inconsistency preserved deliberately:** `DVariable` quotes its
`:name`, `DVarRef` does not. Both were transcribed as-is. Unifying them would be
an output-visible change and wants its own commit, not a conversion turn.

Pinned in `s_dvarref_v` (`xo-expression2/utest/printable_render.test.cpp`), six
cases over binding kind (local / global) × link (0 / 2) × margins 200 / 30 / 20
/ 12. A sentinel `Binding` is **not** among them and cannot be: `DVarRef::make`
builds its binding through `Binding::relative`, which asserts on a sentinel
(`Binding.cpp`), so `"{path}"` is unreachable by construction. Margin 12 breaks
both field values at once, so the known field-value column divergence (legacy 4,
ppsink 3) appears twice in one render.

Mutation-checked four ways — struct name, adding `quot()` to `:name`, renaming
`:path` to `:binding`, and the null-name guard — each fails exactly the cases it
should.

### Colour: ppsink's gate now defaults ON (RC's call, 2026-08-09)

Found while pinning `DRuntimeError`: legacy defaults colour ON
(`xo/indentlog/print/tag_config.hpp:29`), ppsink defaulted it OFF, so **every
call site moved from `toppstr2` to `toppstr` would silently lose colour** and
need a line of its own to put it back. Both are process-wide statics rather
than fields of either config object, so this is a default rather than a protocol
divergence.

`xo::pp::color_config::color_enabled` now defaults **true**
(`xo-ppsink/include/xo/ppsink/color.hpp`). False was the right default while
ppsink was new — adopting it could not then change any existing output — and is
the wrong one now that adoption is the whole project. RC: "may have to rebase
some surrounding unit tests, but will simplify phase D".

Rebasing cost, measured: **6 assertions in 2 files**, all pinning scope-banner
text that now carries the nesting-level colour (`xterm 153`) —
`xo-ppsink/utest/scope.test.cpp`, `xo-indentlog2/utest/scope.test.cpp`. Full
sweep otherwise unchanged.

Two things came with it:

- **`xo::pp::color_enabled_guard`** (`color.hpp`) — RAII over the gate, distinct
  from `color_guard`, which emits escapes. Anything pinning rendered text says
  `color_enabled_guard no_color(false);`. Preferred to assigning the global,
  because set-then-restore leaves it flipped for every later test if an
  assertion fires in between — a hazard the existing tests already worked
  around by hand ("reset BEFORE asserting").
- **A test for the default itself** (`color-enabled-by-default`). Every other
  test in that file turns colour off in order to pin text, so without it the
  default could be flipped back and the suite would stay green. That is
  presumably how it stayed false unnoticed.

**Reopened by `522799d7`, for the no-config entry point** (found 2026-08-09,
after that commit). `toppstr(x)` — the overload with no `PpConfig` — used to
delegate to `toppstr(PpConfig(), x)`, whose default-constructed `PpStyle` is
coloured. It now delegates to `toppstr(PpConfig::plain(), x)`, which is not.
Measured against the installed headers:

```cpp
toppstr(tag("k", x))                     // ":k 1"                  -- was coloured
toppstr(PpConfig(), tag("k", x))         // "\e[38;5;245m:k\e[0m 1"
toppstr(PpConfig::colored(), tag("k",x)) // "\e[38;5;245m:k\e[0m 1"
```

So the exact failure mode this section was written to prevent — a call site
moving from `toppstr2` to `toppstr` and silently losing colour — is live again
for every caller that does not pass a config. No test caught it because, as
recorded above, every test in `toppstr.test.cpp` deliberately renders without
colour in order to pin text; `color-enabled-by-default` guards
`color_config::color_enabled`, not `PpConfig`'s style member.

**Settled the same day (RC, 2026-08-09): the decision above stands unnarrowed.**
The no-config overload now uses `PpConfig::colored()`, so `toppstr(x)`,
`toppstr(PpConfig(), x)` and `toppstr(PpConfig::colored(), x)` agree again. What
was missing was not the default but a test *naming* it: an overload that picks
its own config is constrained by nothing else in the file.
`toppstr-no-config-is-colored` now pins it, and fails if the overload is
switched back. See `.xo-backlog/xo-indentlog2/issues/05` — fixed.

Same shape as `color-enabled-by-default` above, and for the same reason: every
other test in that file renders colourless in order to pin text, so a default no
test names is a default that can flip silently.

### `PpStyle`: colours become per-sink, and get legacy's values (2026-08-09)

The gate alone did not close the phase-D gap: `tag_config::tag_color` was
`none()`, so `:name` stayed uncoloured. Legacy had two conflicting answers and
no test for either —

| legacy path | colour |
|---|---|
| tag ostream inserters (`tag_config::tag_color`) | `xterm(245)`, grey |
| `pretty_struct` (`print/pretty.hpp:407,425`) | `yellow()`, hardcoded, `// tag_config::tag_color` left commented beside it |

— and ppsink had one knob for both. RC: the yellow began as a diagnostic, but
the tag/field distinction turned out worth keeping, so it stayed after its
original motivation went away. **Inherited and chosen, not inherited by
default.**

So both values are kept, and the knob is split in two. The obvious home,
`PpConfig::tag_color`, does not work: the consumers are
`Prettifier<tag_impl>` (`xo-ppsink/.../tag.hpp`) and `Prettifier<field_impl>`
(`xo-ppsink/.../pretty_struct.hpp`), which are handed only a `PpSink &`, and
`PpConfig` is in xo-indentlog2 —

```bash
xo-deps --why=xo-indentlog2:xo-ppsink -q
#   xo-indentlog2 -> xo-ppsink
```

— one subsystem *above*. Independently, `FlatSink` has no `PpConfig` at all, so
a config-only home leaves every flat render with no answer.

Hence **`PpStyle`** (`xo-ppsink/include/xo/ppsink/PpStyle.hpp`): the values at
PpSink level, reached through `PpSink::style()`, settable per sink with
`PpSink::with_style()`. `PpConfig` carries one and `PrettySink` installs it on
itself at construction — the only thing connecting the two, and therefore
tested (`xo-indentlog2/utest/toppstr.test.cpp`, `toppstr-carries-style`;
dropping the `with_style` line fails 3 assertions).

Contents: `tag_color` (grey), `struct_tag_color` (yellow), and
`tag_value_offset` — the last folded in from `tag_config::value_offset` and
renamed for its context, since it is the offset *below a tag name*, not a
general one. It is what produced the 3-vs-4 divergence above, now configurable
rather than a global.

`tag_config` is **gone**, not aliased: two names for one setting is how half a
program configures the other half's copy.

**This is the hook `xo-ppsink/issues/07` requires** — a sink-level place to
reach presentation state. A nested context becomes a stack behind `style()`
rather than a second mechanism.

#### Rebasing cost: the number that mattered

The gate flip alone cost 6 assertions. Giving the colours non-`none` defaults
cost **~60 assertions across 8 subsystems** — every test in the tree that pins a
rendered struct or tag: xo-ppsink, xo-indentlog2, xo-reflect, xo-ratio,
xo-alloc, xo-expression, xo-interpreter, xo-webutil. Ten times the estimate the
design was agreed on, which is worth stating plainly.

It was not paid per assertion. **Each utest main installs `PpStyle::plain()`
once** (8 lines total): unit tests pin readable text, and 60 expectations
containing raw SGR escapes would be unmaintainable. A test wanting colour asks
locally, via `default_style_guard` or `sink.with_style()`.

That leaves the real defaults exercised nowhere — they could be reverted to
`none()` and the suite would stay green, which is exactly how the colour gate
came to be `false` unnoticed. So `xo-ppsink/utest/PpStyle.test.cpp` is the one
place they are asserted, and it constructs the styles it checks rather than
reading the ambient default. Mutation-checked both ways: changing either
default fails 3 assertions.

**`color_enabled_guard` did not survive — superseded same day.** It was written
to reach *scope* colours (`nesting_level_color`, `code_location_color`, the
function entry/exit colours), which are `scope_config`, not `PpStyle`, and stay
that way: a PrettySink should know nothing about `xo::pp::scope`. But the gate
it moved, `color_config::color_enabled`, was itself a process-wide global, and
RC took the next step rather than leaving two mechanisms:

- `bool color_enabled` moves into `PpStyle`
- `color_guard` reads `sink.style().color_enabled` — it already had the sink,
  and every emission site goes through it (`scope.hpp:226,276`,
  `scope.cpp:55,85`)
- `color_config` and `color_enabled_guard` are **deleted**, on the same argument
  that retired `tag_config`: two names for one setting

The payoff is that the retirement *removes* call sites rather than replacing
them. All 9 `color_enabled_guard` uses (8 in `xo-ppsink/utest/scope.test.cpp`,
1 in `xo-indentlog2/utest/scope.test.cpp`) can be deleted outright, because the
utest mains already install `PpStyle::plain()` and every sink copies the default
at construction — **provided `plain()` clears `color_enabled`**, which is the
one detail that makes the deletion work. `plain()` clearing only the two colour
values leaves scope banners coloured in tests.

Behaviour change worth noting beyond the refactor: colour enablement becomes
per-sink, captured at construction, exactly like the colours. A "not a tty →
no colour" check must therefore run before any sink is built, or set
`PpStyle::default_style().color_enabled` and build after. One rule instead of
two, which is the point.

**Unrelated bug found next door, not fixed:** legacy `print_fg_color_on` emits
`\033[31;<code>m` for `color_encoding::ansi`
(`xo-indentlog/include/xo/indentlog/print/color.hpp:80`) — a hardcoded `31;`
(red) ahead of the requested colour, so legacy `yellow()` is really "red, then
yellow". ppsink emits `\033[<code>m` (`xo-ppsink/src/ppsink/color.cpp:27`),
i.e. correctly. Anyone diffing raw escapes between the two stacks will meet
this; it is legacy-only and dies with it.

## Phase E checklist — things to delete, not just `pretty_deprecated`

Recorded as they accumulate, because each is easy to leave behind:

- `pretty_deprecated` from `xo-printable2/idl/Printable.json5`, then regenerate
- the 55 hand-written `pretty_deprecated` implementations
- **`render_deprecated()` from every phase C test**, and with it the
  `REQUIRE(modern == legacy)` line — the absolute assertion is what remains
- the `#include <xo/indentlog/print/ppstr.hpp>` those tests carry for
  `toppstr2`, which is the last xo-indentlog dependency in the cluster's tests
- `xo-printable2/include/xo/printable2/detail/ppdetail_Printable.hpp` and its
  entry in `user_hpp_includes`
- `<xo/indentlog/print/ppindentinfo.hpp>` from the IDL's `includes:`, and the
  `ppindentinfo` entry from its `types:`

Only after all of that does `xo-deps --why=xo-printable2:xo-indentlog` return
rc=1, which is the milestone's actual goal.

### Regeneration is manual, and its output is committed

```bash
grep -rc 'manual target; generated code committed' --include=CMakeLists.txt xo-*/
#   152 such targets across 11 subsystems, xo-reader2 alone 44
git ls-files --error-unmatch \
    xo-expression2/include/xo/expression2/variable/IPrintable_DVariable.hpp   # tracked
```

Two consequences. Each phase that touches the IDL is a deliberate regeneration
across 11 subsystems producing a large committed diff — reviewable only by
knowing it is generated. And a stale generated file does not self-heal on
build, so "did I regenerate everything" is a real question: check by rebuilding
from clean rather than by inspection.

The per-type IDLs reference the facet IDL rather than restating it
(`facet_idl: "idl/Printable.json5"` plus `FACET_PKG xo_printable2` in the
CMake target), so there is exactly one copy of the method list and no
per-consumer IDL edits. Confirmed: `find xo-*/ -name 'Printable.json5'` returns
one path.

## The precedent to copy

`.xo-backlog/xo-expression/issues/01-ppsink-migration-pilot.md` did this exact
conversion on the older expression/reader stack, deliberately first so this
cluster benefits. It records: the return value is vestigial and becomes `void`;
~2/3 of conversions are mechanical `pretty_struct` substitutions; capture a
rendered baseline BEFORE converting, since it is the only way to tell a dropped
field from a layout convention.

`.xo-backlog/xo-alloc/issues/01` (done 2026-08-09) is the more recent precedent
for the bridging shape, and contributes two warnings: a `Prettifier`
specialization must be constrained over the hierarchy rather than written for
the base type, or derived types silently miss it; and the audit for stale
callers is the build, not grep — deleting the old method turns every one into a
compile error, which is how a fourth and fifth subrepo surfaced there.

**Files:**
- `xo-printable2/idl/Printable.json5` — the IDL; phases A, B and E are edits here
- `xo-facet/README.md` — how genfacet is invoked, and the `xo_add_genfacet`
  CMake form
- `xo-facet/codegen/*.j2` — the templates, if the generated shape itself needs
  to change
- the 111 `DFoo.*pp` files, concentrated in xo-reader2 (48)

**Done when:**
- no `ppindentinfo` remains in the cluster, generated or hand-written
- `xo-printable2` no longer declares an `xo-indentlog` dependency
- rendered output is unchanged, pinned by baselines captured before phase C

## Notes

Regenerating is a build target, not a script to run by hand:

```bash
cmake --build path/to/build -- xo-printable2-facet-printable
```

168 `xo_add_genfacet` invocations exist across 11 subsystems, so confirm what a
rename regenerates before assuming it is only printable2's own scaffolding.
