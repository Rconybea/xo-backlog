# 02 — a DTypename with a null type_ ABORTS the process, on both print protocols

Status: diagnosed
Type: bug

`DTypename::pretty` and `DTypename::pretty_deprecated` both do

```cpp
auto type_pr = type_.to_facet<APrintable>();
```

For a **non-null** `type_` this throws, deliberately — see
`.xo-backlog/xo-type/issues/01` and the `DTypename-render` test; the throw is a
standing red test for xo-type's missing `APrintable` facet, and RC's call
(2026-08-11) is to keep it.

For a **null** `type_` it does something else, and nothing about that is
deliberate:

```
fatal: attempt to call uninitialized IPrintable_Any method
terminate called without an active exception
SIGABRT
```

`to_facet<APrintable>()` on an empty `obj<AType>` returns an **empty**
`obj<APrintable>` rather than throwing. The empty handle then reaches
`pretty_struct`, and rendering it calls through an uninitialized vtable slot.

## Why this is worse than the throw it sits next to

An exception is catchable, reportable, and testable — `DTypename-render` pins it
with `REQUIRE_THROWS_AS`. This is an `abort()`: it takes the process down
uncatchably, Catch2 cannot recover from it, and a `REQUIRE_THROWS` cannot assert
it. Confirming it needs a death test or a subprocess.

Measured 2026-08-11, both protocols, independently:

```bash
# xo-expression2/utest, temporary probe: DTypename::make(mm, name, obj<AType>())
#   pretty path skipped, deprecated only  -> SIGABRT
#   both paths                            -> SIGABRT
```

**Pre-existing**, not introduced by the ppsink conversion — the deprecated
printer aborts with the pretty path removed from the probe entirely. The
conversion reproduced it faithfully, which is the correct outcome for a pure
refactor and is why this is a separate ticket rather than a fix folded into it.

## Is it reachable?

Unknown, and worth settling before choosing a fix. `DLocalSymtab::append_type`
takes `obj<AType> type` and does not check it:

```cpp
obj<AGCObject> tname = DTypename::make(mm, name, type);
types_->push_back(mm, tname);
```

so a caller passing an empty `obj<AType>` produces one. Whether the reader ever
does is not established here.

## The wider question this is an instance of

The specific fault is not really DTypename's: **`to_facet<A>()` on an empty
`obj<>` yields an empty `obj<A>` that aborts on use, rather than failing at the
lookup.** Anywhere a possibly-empty handle is converted and then rendered has
the same shape. This ticket fixes the one observed instance; whether
`obj<>::to_facet` should reject an empty source is a facet-layer question and
bigger than this.

**Files:**
- `xo-expression2/src/expression2/DTypename.cpp` — both printers
- `xo-expression2/include/xo/expression2/DLocalSymtab.hpp` — `append_type`, the
  unchecked entry point
- `xo-facet/include/xo/facet/FacetRegistry.hpp` — `variant` vs `try_variant`
- `.xo-backlog/xo-type/issues/01-no-aprintable-facet.md` — the *non*-null case,
  which is deliberate

**Done when:**
- reachability from the reader is established or ruled out
- a null `type_` produces something diagnosable — a throw, or a rendering — not
  an abort
- there is a test for it, which means deciding how to assert an abort (death
  test) or eliminating the abort first and asserting the replacement

## Notes

Found while converting `DTypename` to ppsink: the first OBSERVE probe used a
null `type_` on the assumption that it would be the one *renderable* case, since
the populated one was known to throw. It aborted instead — so `DTypename` has
**no renderable case at all**, on either protocol.

That has a consequence worth stating plainly: `DTypename`'s struct shape — the
field names `:name` / `:type`, their order, the quoting of the name — is pinned
by **no test**, and cannot be while every case terminates. When xo-type gains
`APrintable`, all of it becomes newly-exercised code with no coverage behind it.
