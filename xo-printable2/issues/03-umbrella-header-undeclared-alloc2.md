# 03 — Printable.hpp includes alloc2, which printable2 neither declares nor may depend on

Status: open
Type: bug

`xo-printable2/include/xo/printable2/Printable.hpp` — the convenience header a
consumer is meant to include — pulls `<xo/alloc2/Allocator.hpp>`:

```bash
grep -n 'alloc2' xo-printable2/include/xo/printable2/Printable.hpp
grep -n 'user_hpp_includes' -A 2 xo-printable2/idl/Printable.json5   # where it comes from
```

Two things are wrong with that, and they are separable.

**1. Undeclared.** printable2's only declared dependency is xo_facet:

```bash
grep -n 'xo_dependency' xo-printable2/src/printable2/CMakeLists.txt
xo-deps --why=xo-printable2:xo-alloc2 -q; echo "exit=$?"   # 1, no path
```

**2. Undeclarable.** alloc2 is levelled ABOVE printable2, so the edge cannot be
added without inverting them:

```bash
grep -n '^xo-printable2$\|^xo-alloc2$' xo-cmake/etc/xo/subsystem-list
#   18:xo-printable2
#   19:xo-alloc2
```

Nothing in printable2's own headers appears to need it — grep for `alloc2`,
`Allocator` or `mm::` across `include/xo/printable2/detail/` returns nothing.

## Why it has gone unnoticed

printable2's own sources include the `detail/` headers, not the umbrella one, so
its build and its nix package are both green. Every consumer of the umbrella
header today sits above alloc2 anyway:

```bash
grep -rln 'printable2/Printable.hpp' xo-*/include xo-*/src xo-*/utest | grep -v '/\.build/'
#   as of 2026-09-12: xo-expression2 (subsystem 32), and similar
```

so the include resolves by accident of what else is on their include path.

Found 2026-09-12 from xo-reflectable2's utest, which is the first consumer BELOW
alloc2 to reach for the header. It fails outright:

```
xo-printable2/include/xo/printable2/Printable.hpp:22:10:
  fatal error: xo/alloc2/Allocator.hpp: No such file or directory
```

Worked around there by including `detail/APrintable.hpp`,
`detail/IPrintable_Any.hpp`, `detail/IPrintable_Xfer.hpp` and
`detail/RPrintable.hpp` directly — which is the tell: if the umbrella header is
avoidable, the include in it is doing nothing for printable2 itself.

The IDL comment records the include's history (hand-added to the GENERATED file
by `b1add3bb`, silently dropped by the next regeneration, then moved into
`user_hpp_includes` where regeneration preserves it). What it does not record is
WHY a facet header needs an allocator, which is the question this ticket is
really asking.

**Done when:**
- either the include is gone from `user_hpp_includes`, or printable2's levelling
  and declared dependencies account for it
- `xo-reflectable2/utest/FopTdx.test.cpp` can go back to
  `<xo/printable2/Printable.hpp>` — the check that a below-alloc2 consumer can
  use the convenience header
- whichever way it goes, the reason is written down, since the include has
  already survived one silent round-trip through the generator
