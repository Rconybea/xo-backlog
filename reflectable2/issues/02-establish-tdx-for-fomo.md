# 02 — a fomo object reflects as a pointer to its representation

Status: done 2026-09-12
Type: feature
Milestone: reflectable2

`Reflect::require<obj<AFacet,DRepr>>()` should yield a usable `TypeDescr`. Today
it falls through to the atomic default, so a fomo object is a leaf with no way
in:

```bash
sed -n '20,35p' xo-reflect/include/xo/reflect/Reflect.hpp    # EstablishTdx primary -> AtomicTdx::make()
```

## Shape

Specialize `EstablishTdx<obj<AFacet,DRepr>>` in xo-reflectable2, reporting
**`mt_pointer` with one child, the representation**. That is what a fomo object
is — a facet plus a data pointer — and it means printjson needs no new case:
its `print_generic_pointer` already walks `n_child()`/`get_child(0)`.

```bash
sed -n '55,75p' xo-printjson/src/printjson/PrintJson.cpp     # print_generic_pointer
```

The same shape reflect already uses for `rp<Repr>`, which is the analogy to
follow:

```bash
ls xo-reflect/include/xo/reflect/pointer/
```

When `DRepr` is concrete (not `DVariantPlaceholder`), the child's `TypeDescr` is
just `Reflect::require<DRepr>()`. The erased case — `vt<AFacet>`, i.e.
`obj<AFacet, DVariantPlaceholder>` — is deferred to `03`; until then it may
report zero children rather than guessing.

Note which case is routine: D-types hold erased fomo members as a matter of
course, so the static case is the exception, not the rule.

```bash
grep -n 'obj<AGCObject>' xo-object2/include/xo/object2/DDictionary.hpp \
                         xo-object2/include/xo/object2/DList.hpp | head
```

**Files:**
- Add: `xo-reflectable2/include/xo/reflectable2/FopTdx.hpp` (or similar) with
  the `EstablishTdx` specialization
- Test: `xo-reflectable2/utest/`

**Done when:**
- `Reflect::require<obj<AFacet,DRepr>>()` reports `mt_pointer`, `n_child() == 1`,
  and `child_tp(0).td()` is `Reflect::require<DRepr>()`
- printjson emits JSON for a statically-typed fomo object handed to
  `print_tp()` as a hand-built `TaggedPtr` — no new printjson entry point
  needed to see output at this stage
- `xo-reflect` is unmodified; `git diff --stat xo-reflect/` is empty

## Outcome (2026-09-12)

Done. A statically-typed fomo object now renders:

```
{"_name_": "DFopPoint", "x": 1.5, "y": -2.5}
```

identical to printing the representation directly -- which is the assertion the
test makes, rather than comparing against a literal, so the claim survives a
change to struct formatting.

`FopTdx<AFacet,DRepr> : PointerTdx` plus the `EstablishTdx` specialization, in
`xo-reflectable2/include/xo/reflectable2/FopTdx.hpp`. Named for "faceted object
pointer" -- NOT "Fomo", which names the whole collection of features rather
than this one pointer-shaped thing. The pointer itself is `obj<AFacet,DRepr>`
today and is expected to become `fop<AFacet,DRepr>`; when that lands, these
names already agree. One class rather than
two, because `03` adds `most_derived_self_tp()` to this same type; splitting
would give one concept two names. Erased and concrete differ by `if constexpr`
on `c_erased`, so there is no second code path to keep in step.

### Correction to this ticket's test plan

The ticket said **Test: `xo-reflectable2/utest/`** and also asked that printjson
emit JSON. Those conflict: xo-printjson is subsystem 56, xo-reflectable2 is 22,
so a printjson test inside reflectable2's utest is an upward dependency that
breaks its standalone build. Recorded rather than quietly fixed, because the
same trap catches any ticket that wants a low subsystem tested through a high
consumer.

Split instead:

- `xo-reflectable2/utest/FopTdx.test.cpp` -- reflection assertions
- `xo-printjson/utest/FopJson.test.cpp` -- the JSON, with a TEST-ONLY
  dependency on xo_reflectable2. The library edge is still `04`'s to add.

### Falsified, not just observed

Removing the `EstablishTdx` specialization and rebuilding turns 3 of the 4
reflection cases red -- `metatype()` reports 1 (`mt_atomic`) instead of 2
(`mt_pointer`), since the primary template falls through to `AtomicTdx`. Worth
doing: without it, "the specialization is what makes this work" is an
assumption.

The case that still passed under falsification is `empty-fomo-object-has-no-child`
-- an atomic reports zero children too. It discriminates nothing on its own and
is kept only as the pair to the non-null case.

`erased-fomo-object-reports-no-child-yet` pins the erased behaviour so `03` has
a red test to turn green, rather than changing behaviour nothing was watching.

### Nix packaging, which neither 01 nor 02 mentioned

A new subsystem is not finished at `xo-build --sweep`: `pkgs/xo-reflectable2.nix`,
plus lines in `xo.nix` and `ci.nix`, are also needed, and xo-printjson's nix
expression needed the test-only input as well. CONVENTIONS calls nix the only
check that exercises an installed package config as a real consumer would, so a
subsystem missing from it is invisible to the check that matters most.

```bash
nix-build ci.nix -A xo-reflectable2 --no-out-link
nix-build ci.nix -A xo-printjson --no-out-link
grep -n 'utest.reflectable2' <log>   # confirm the CHECK phase ran, not just the build
```

Both green, and the check phase really ran -- worth grepping for, since a nix
expression missing `-DENABLE_TESTING` builds green while running zero tests.

**Verified:** sweep green, stage 2 `38 ok, 32 with no tests` -> `39 ok, 31 with
no tests`; `xo-reflectable2 ok (utest)` rather than `ok (utest:no-tests)`. The
attempted total does NOT move, because no subsystem was added.
