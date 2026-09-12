# 02 — genfacet emitted a hardcoded facet name in every generated _Any.cpp

Status: fixed 2026-09-12
Type: bug

`iface_facet_any.cpp.j2` carried the literal text `IAllocator_Any` inside the
`_fatal()` comment instead of the `{{iface_facet_any}}` placeholder, so every
facet in the tree claimed to be the allocator facet:

```bash
grep -rn 'uninitialized IAllocator_Any' xo-*/src --include=*.cpp | grep -v xo-facet/src
# empty after the fix; 18 files before it
```

Cosmetic — a comment, not code — but it misdirects a reader landing on a real
`_fatal()` at runtime, which is exactly when they are least able to afford it.

## Cause

The template was derived from `xo-facet/src/facet/IAllocator_Any.cpp`, which is
HAND-written, not generated — it carries an author line, `LCOV_EXCL_START`, and
an extra comment line the template lacks. Three files in xo-facet are in that
category and were correctly left alone:

```bash
find xo-* -name 'I*_Any.cpp' -not -path '*/.build*' | wc -l   # generated + hand-written
grep -l 'mode: *"facet"' xo-*/idl/*.json5 | wc -l             # the generated ones
```

The difference between those two counts is the hand-written set. When the
template was lifted, one occurrence of the name was parameterized and the other
was not; nothing checks a generated comment, so it survived every facet added
since.

## Fix

One-token change to the template, then regenerate. Regeneration is per
subsystem because IDL paths are relative to it:

```bash
GF=$PWD/xo-facet/codegen/genfacet
for idl in $(grep -l 'mode: *"facet"' xo-*/idl/*.json5); do
    sub=${idl%%/*}; ( cd $sub && $GF --input ${idl#*/} )
done
```

**Confirmation that nothing else drifted:** the regeneration diff is exactly one
line per file. Generated files are checked in here, so a template edit that
changed anything more would have shown up as a larger diff — worth checking
rather than assuming, since a generator touched 19 IDLs at once.

Verified by `xo-build --sweep` green.
