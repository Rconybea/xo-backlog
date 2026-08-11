# 01 — Prettifier<Binding>, and the other types taking ppsink's operator<< fallback

Status: open
Type: task
Milestone: ostream-containment

Raised by RC 2026-08-10, from the `DVarRef` conversion in
`.xo-backlog/xo-printable2/issues/01-aprintable-pretty-ppsink.md`.

## What was found

`DVarRef::pretty()` renders `:path`, a `Binding`. `Binding` has **no
`Prettifier<>` and no `ppdetail<>` — only an `operator<<`**
(`xo-expression2/include/xo/expression2/Binding.hpp:58`), so it takes ppsink's
third and last dispatch branch:

```cpp
/* xo-ppsink/include/xo/ppsink/pretty.hpp:34 */
if constexpr (has_prettifier<T>) { ... }               /* 1: native */
else if constexpr (std::is_convertible_v<T, std::string_view>) { ... }  /* 2: leaf */
else { auto ins = sink.stream_open(1); ins << x; }     /* 3: fallback */
```

That branch is not an oversight in ppsink — it is documented, deliberately, as
the last resort: *"This is the ONLY path that needs `<ostream>` (at the point of
instantiation); prefer a `Prettifier<T>` specialization so a type never lands
here"* (`pretty.hpp:29-32`).

It works. Both protocols render `{path:0:3}` byte-identically at every margin,
pinned in `s_dvarref_v`
(`xo-expression2/utest/printable_render.test.cpp`). This ticket is not a bug
report.

## Why do it anyway

**1. The fallback is silent.** A type with an `operator<<` and no
`Prettifier<>` compiles and renders flat, with no diagnostic. Contrast
`TypeRef`, which had no `operator<<` and so failed to *compile* until given a
`Prettifier<>` — that is the loud case, and it is the lucky one. Nothing tells
you a type took branch 3 except pinning its output and looking.

**2. It drags `<ostream>` into the object model.** `Binding.hpp:9` includes
`<iostream>` — not `<ostream>`, not `<iosfwd>` — and is included by six headers:

```bash
grep -rn 'Binding.hpp' --include=*.hpp --include=*.cpp xo-*/ | grep -v '/\.build/'
#   DGlobalSymtab.{hpp,cpp}, DLocalSymtab.hpp, DVariable.hpp,
#   symtab/ASymbolTable.hpp, symtab/ISymbolTable_Xfer.hpp
```

`<iostream>` additionally instantiates `std::ios_base::Init` in every
translation unit that sees it. With a `Prettifier<Binding>`, `Binding.hpp` could
fall to `<iosfwd>` — or drop the stream dependency entirely if `print()` goes
with it. **`Binding::print` has exactly one caller**, the `operator<<` two lines
below it:

```bash
grep -rn '\.print(\|->print(' --include=*.cpp --include=*.hpp xo-expression2/ | grep -v '/\.build/'
#   xo-expression2/include/xo/expression2/Binding.hpp:59:  x.print(os);
```

So retiring both is a live option rather than a guess — though whether to keep
an `operator<<` for newcomers and tests is the same question
`xo-webutil/include/xo/webutil/webutil_ostream.hpp` already answered once, and
should be answered the same way.

**3. The regression check already exists.** The conversion is output-neutral by
construction — a `Prettifier<Binding>` can `sink.put()` exactly the bytes
`Binding::print` emits — and `s_dvarref_v` already pins those bytes through both
protocols. So this is a refactor with its test written first, unusually.

## The other four

Same shape, same silence, elsewhere in the facet cluster. Measured 2026-08-10:

```bash
# NB match the PARAMETER, not the return type -- xo's style puts the return
# type on its own line, and the earlier `ostream & operator<<` form undercounted
# tree-wide by 6x.  See the CORRECTED section of the ostream-containment
# milestone.
grep -rn 'operator<< *( *std::ostream' --include=*.hpp \
  xo-expression2/include xo-reader2/include xo-interpreter2/include | grep -v '/\.build/'
```

**That table is the facet cluster only, and is now known to be incomplete** —
re-run the corrected grep above before relying on it; the reader2 count alone
went from 3 files to 19 once the pattern was fixed.

| type | file |
|---|---|
| `Binding` | `xo-expression2/include/xo/expression2/Binding.hpp:58` |
| `exprseqtype` | `xo-reader2/include/xo/reader2/DToplevelSeqSsm.hpp:29` |
| `parser_result_type` | `xo-reader2/include/xo/reader2/ParserResult.hpp:28` |
| `ParserResult` | `xo-reader2/include/xo/reader2/ParserResult.hpp:106` |
| `ParserStack *` | `xo-reader2/include/xo/reader2/ParserStack.hpp:79` |

The two enum-like ones (`exprseqtype`, `parser_result_type`) probably want to
stay flat and only need the ostream dependency removed. **`ParserResult` and
`ParserStack` are the interesting ones**: they are structures, so branch 3
renders them as a single opaque token with no break points — which is precisely
the flattening this migration exists to undo. Unverified whether that matters in
practice; check what they actually render at a narrow margin before deciding.

Those four are not this ticket's work. They reach their own phase C turn in
`.xo-backlog/xo-printable2/issues/01`, and the point of listing them here is
that the question will arrive four more times and should get the same answer.

## A sixth, found 2026-08-10, with a wider blast radius than `Binding`

`xo::reflect::typeseq` — `xo-reflectutil/include/xo/reflectutil/typeseq.hpp:115`.
Found during the `DConstant` conversion, whose printer renders two of them; it
is also the type whose multi-line inserter exposed the counting bug in
`.xo-backlog/milestones/ostream-containment.md`.

It wants a `Prettifier<>` more than `Binding` does, on three counts:

- **The implementation is now trivial.** `typeseq` wraps a single `int32_t`
  (`seqno()`), and its inserter is `s << x.seqno()`. Since
  `.xo-backlog/xo-ppsink/issues/09-scalar-prettifiers.md` there is a
  `Prettifier<>` for every integer width, so this is `sink.pp(x.seqno())` — one
  line, output-identical, and already pinned by `s_constant_v`.
- **`typeseq.hpp` includes `<iostream>`, not `<ostream>`** (`:10`) — so every
  consumer instantiates `std::ios_base::Init`.
- **xo-reflectutil is used by 49 subsystems**, verified:
  ```bash
  xo-deps --users-of=xo-reflectutil --format=names -q | wc -l
  ```
  which is very likely the widest `<ostream>` propagation in the tree. `Binding`
  reaches six headers; this reaches nearly everything.

### The decision it forces: xo-reflectutil's first dependency

`Prettifier<>` lives in xo-ppsink, and **xo-reflectutil currently depends on
nothing**:

```bash
xo-deps --deps-of=xo-reflectutil --format=names -q   # prints nothing
xo-deps --why=xo-ppsink:xo-reflectutil               # exit 1 -> no cycle either way
xo-deps --deps-of=xo-ppsink --format=names -q        # xo-ppsink xo-timeutil
```

So `xo-reflectutil -> xo-ppsink` is legal — ppsink is nearly as low, depending
only on itself and xo-timeutil — but it would be the first edge out of the
bottom of the graph, and that is a levelization decision rather than a
refactoring detail.

**There is a precedent, and it is close.** The tree's `<thing>_pp.hpp`
convention already puts a `Prettifier<>` in a separate header so the base header
carries no printing vocabulary — four instances today:

| header | subsystem's deps |
|---|---|
| `xo-refcnt/.../Refcounted_pp.hpp` | ppsink, reflectutil, timeutil |
| `xo-arena/.../span_pp.hpp` | ppsink, randomgen, reflectutil, timeutil |
| `xo-reflect/.../TypeDescr_pp.hpp` | ppsink, refcnt, reflect, subsys, testutil, timeutil |
| `xo-ratio/.../ratio_pp.hpp` | (ten, incl. ppsink) |

`xo-refcnt` is one level above reflectutil and already carries a ppsink
dependency for exactly this purpose. So the shape is established; what is new is
doing it at the very bottom.

Three options, in the order I would consider them:

1. **`xo-reflectutil/include/xo/reflectutil/typeseq_pp.hpp`**, reflectutil
   gaining a ppsink dependency. Matches the convention; costs one graph edge
   from the bottom.
2. **Put `Prettifier<typeseq>` in a consumer that already has both** — xo-facet
   or xo-printable2. No graph change, but the Prettifier is then somewhere
   nobody would look for it, and consumers that do not include that subsystem
   silently stay on the fallback. That last part is the same silent-divergence
   trap this ticket exists to close.
3. **Leave it.** The rendering is correct; only the ostream round-trip and the
   `<iostream>` propagation are at stake.

Unverified either way: whether anything actually depends on `typeseq`'s
`operator<<` outside printers — worth a grep before removing it.

## Suggested approach

Do `Binding` alone first, as its own commit, after phase C rather than during —
it changes no output, so it does not belong in a conversion turn where the diff
is supposed to be reviewable as "did the rendering change".

**Files:**
- `xo-expression2/include/xo/expression2/Binding.hpp:9,58` — the `<iostream>`
  include and the `operator<<`
- `xo-expression2/src/expression2/Binding.cpp` — `Binding::print`, the format to
  preserve exactly
- `xo-expression2/utest/printable_render.test.cpp` — `s_dvarref_v`, the pin

**Done when:**
- `Prettifier<Binding>` exists and `s_dvarref_v` passes unchanged
- `Binding.hpp` no longer includes `<iostream>`
- a decision is recorded on whether `operator<<` stays as an opt-in convenience
  (per the `webutil_ostream.hpp` precedent) or goes

## Notes

**The general question — how do you *find* the types silently taking branch 3 —
now has an answer, and it is structural rather than a tool.** RC's, 2026-08-10:
you are done when every `operator<<` for an xo type lives in a
`*_ostream.hpp` header and no non-test xo header includes one. Then "which types
stream?" is a directory listing rather than a grep with known false positives
(a type may also have a `Prettifier<>`) and false negatives (an inherited or
templated inserter does not match a textual search).

That is `.xo-backlog/milestones/ostream-containment.md`, measured at 47 headers
remaining across 21 subsystems — deliberately a **separate** milestone from
`ppsink-migration`, which it is downstream of rather than part of. `Binding` is
this ticket and is one of the 47; the other four types listed above are also in
that count.
