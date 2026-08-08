# 02 — an empty block `{ }` segfaults the reader

Status: open
Type: bug

`Sequence`'s constructor computes its value type from the last element without
checking that there is one:

```cpp
// xo-expression/include/xo/expression/Sequence.hpp:14
Sequence(const std::vector<rp<Expression>> & xv)
    : Expression(exprtype::sequence,
                 xv[xv.size() - 1]->valuetype()),   // <- underflows when empty
      expr_v_(xv) {}
```

On an empty vector `xv.size() - 1` is `SIZE_MAX` (unsigned), so this indexes
far out of bounds and dereferences whatever it finds.

## Reachable from ordinary source, not just from the API

Measured 2026-08-08 — this is **not** merely an API-misuse hazard:

```
$ seqprobe 'def foo = lambda (x : f64) { };'
...
+(0) expect_expr_xs::on_rightbrace_token
Segmentation fault (core dumped)          # exit 139
```

Both `Sequence::make` call sites build the sequence when the closing brace is
seen, from whatever was accumulated at that level:

- `xo-reader/src/reader/sequence_xs.cpp:119` — `on_rightbrace_token`
- `xo-reader/src/reader/let1_xs.cpp:135` — `on_rightbrace_token`

So any empty block in Schematika source reaches it. An empty `Sequence` cannot
be constructed at all.

## Found by

Writing `xo-expression/utest/pretty.test.cpp` during the ppsink migration: a
`pretty-sequence-empty` case crashed the test binary. The crash is in
construction, entirely unrelated to printing — the migration just happened to
be the first thing that ever tried to build one. That test case is omitted with
a comment pointing here.

## The decision this needs

What *is* the value type of `{ }`? The constructor cannot answer it, so the fix
depends on the language question:

1. **Reject at parse time** — `{ }` is not a legal expression; the reader emits
   a parse error at `on_rightbrace_token` when `expr_v_` is empty. Safest, and
   keeps `Sequence` honest (it always has a value type).
2. **Give it a unit/void type** — `{ }` is legal and evaluates to unit. Needs a
   unit type to exist in the type system; check `xo-expression/typeinf` before
   assuming it does.
3. **Guard the constructor only** — throw on empty. Turns a segfault into a
   diagnosable error without settling the language question, but leaves `{ }`
   unparseable in a confusing way.

(3) is the cheap stopgap and is strictly better than the current behaviour;
(1) or (2) is the real answer. Worth deciding rather than defaulting.

**Whatever the choice, the constructor should not be able to underflow** — the
`xv[xv.size() - 1]` form is unsafe regardless of who is expected to call it.

**Files:**
- `xo-expression/include/xo/expression/Sequence.hpp:14` — the constructor
- `xo-reader/src/reader/sequence_xs.cpp:119`,
  `xo-reader/src/reader/let1_xs.cpp:135` — the two call sites
- `xo-expression/utest/pretty.test.cpp` — where the omitted test case is noted

**Done when:**
- `def foo = lambda (x : f64) { };` either parses or reports an error; it does
  not crash
- a test covers the empty case, whichever behaviour is chosen

## Notes

`grep -rn "size() - 1\]" xo-expression/include xo-reader/include` finds only
this one site, so it does not appear to be a pattern repeated elsewhere.
