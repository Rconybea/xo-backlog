# 02 — a fomo object reflects as a pointer to its representation

Status: open
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
- Add: `xo-reflectable2/include/xo/reflectable2/FomoTdx.hpp` (or similar) with
  the `EstablishTdx` specialization
- Test: `xo-reflectable2/utest/`

**Done when:**
- `Reflect::require<obj<AFacet,DRepr>>()` reports `mt_pointer`, `n_child() == 1`,
  and `child_tp(0).td()` is `Reflect::require<DRepr>()`
- printjson emits JSON for a statically-typed fomo object handed to
  `print_tp()` as a hand-built `TaggedPtr` — no new printjson entry point
  needed to see output at this stage
- `xo-reflect` is unmodified; `git diff --stat xo-reflect/` is empty
