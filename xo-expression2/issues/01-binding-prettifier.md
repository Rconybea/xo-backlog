# 01 — Prettifier<Binding>, and the other types taking ppsink's operator<< fallback

Status: open
Type: task

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
grep -rn 'std::ostream & *operator<<\|ostream & operator<<' --include=*.hpp \
  xo-expression2/include xo-reader2/include xo-interpreter2/include | grep -v '/\.build/'
```

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

The general question — how do you *find* the types silently taking branch 3 —
has no tool today. The grep above finds declared `operator<<`s, which is a proxy:
it over-reports (a type may also have a `Prettifier<>`) and under-reports (an
inherited or templated inserter would not match). A `static_assert`-based opt-out,
or a compile-time listing of types instantiating the fallback, would be a real
answer; neither is designed.
