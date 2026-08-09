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
interpreter2 → reader2. This is the real work, ~55 types.

**Phase D — switch call sites** from `pretty_deprecated` to `pretty`.

**Phase E — delete** `pretty_deprecated` from the IDL, regenerate, remove the
old implementations.

### The risk this shape carries

`IPrintable_Any` terminates rather than defaulting:

```cpp
/* xo-printable2/include/xo/printable2/detail/IPrintable_Any.hpp:59 */
[[noreturn]] bool pretty(Copaque, const ppindentinfo &) const override { _fatal(); }
```

So after phase B every type has a **fatal** `pretty(PpSink&)` until phase C
reaches it. That is a runtime landmine, not a compile error — unlike
`.xo-backlog/xo-alloc/issues/01`, where the base class carried a working
default and an unconverted subclass simply rendered through the old path.

Two mitigations, and the choice is open:

- keep phases C and D tightly coupled per subsystem, so no call site reaches a
  type that has not been converted; or
- give the new method a real default that bridges to `pretty_deprecated`.
  **Unverified whether that is constructible**: `ppindentinfo` is a two-pass
  fit/emit protocol and `PpSink` is single-pass, and
  `.xo-backlog/xo-expression/issues/01` found the two-pass bodies were the hard
  part of the pilot rather than something mechanically wrappable. Worth an hour
  to establish before committing to the order, because a working bridge would
  make phase C interruptible.

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
