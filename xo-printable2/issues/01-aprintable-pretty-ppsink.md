# 01 — APrintable::pretty(ppindentinfo) -> pretty(PpSink&), across the facet cluster

Status: open
Type: task
Milestone: ppsink-migration

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
nothing, so a baseline diff shows exactly which types are still pending.

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

**So phase B's stubs render nothing**, and the baseline diff is how phase C is
steered: a stubbed type shows as missing output, which makes "what is still
pending" observable rather than tracked by hand.

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
