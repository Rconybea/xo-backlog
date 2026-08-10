# 09 — Prettifier for every integer width, and for bool

Status: fixed 2026-08-10
Type: task

Before this, `Prettifier<>` covered `int`, `double` and `float` and nothing
else. Every other integer width — including `std::uint32_t`, `std::size_t` and
`std::int64_t` — fell through the empty primary template to pretty()'s
`operator<<` fallback: correct output, via an ostream, invisibly.

Raised by RC 2026-08-10, on finding that `DGlobalSymtab`'s four fields are
`std::uint32_t` (`xo-expression2/include/xo/expression2/DGlobalSymtab.hpp:39`)
and would therefore have pinned the fallback rather than a Prettifier, in the
middle of a migration whose point is to stop rendering through ostreams.

## What was measured first

Every scalar's rendering was observed **before** any specialization existed
(`tostr()` through a FlatSink), so that the change could be shown to be
output-neutral rather than assumed to be:

| type | rendered | had a Prettifier? |
|---|---|---|
| `short`, `unsigned short`, `unsigned int`, `long`, `unsigned long`, `long long`, `unsigned long long` | decimal digits | no |
| `std::uint32_t`, `std::size_t` | decimal digits | no |
| `bool` | `1` / `0` | no |
| `char`, `signed char`, `unsigned char` | **`A`** (for 65) | no |
| `std::int8_t(-1)` | one garbage byte | no |

`to_chars` and `operator<<` agree on decimal integers, so widening the
specialization changes nothing. Re-measured after: identical strings,
`has_prettifier` flipped to 1 for the numeric widths and **still 0 for the char
types**.

## The trap

`std::is_integral_v<char>` is **true**. A specialization constrained on plain
`std::integral` would have captured `char`, `signed char` and `unsigned char`
and turned `'A'` into `"65"` in every rendering in xo — and `std::int8_t` /
`std::uint8_t` with them, since those are typedefs for the char types on any
normal platform.

So the constraint is `pp_number_integral`
(`xo-ppsink/include/xo/ppsink/Prettifier.hpp`), which is `std::integral` minus
`bool` and minus all seven character types. Dropping those exclusions is caught
at **compile** time by `static_assert(!has_prettifier<char>)` in
`xo-ppsink/utest/Prettifier.test.cpp`, not merely at test time.

## `bool` renders `1`/`0`, deliberately

`operator<<` prints a bool as `1`/`0` unless `std::boolalpha` is set, and
nothing in xo sets it. Renderings already pinned across the tree contain it —
e.g. `TypeDescr`'s `:complete 1` in
`xo-expression2/utest/printable_render.test.cpp`.

`Prettifier<bool>` therefore **removes the ostream, not the format**. Whether
`true`/`false` would read better is open, and is now a one-line change in one
place rather than a property inherited from `<ostream>` — the same framing
`c_default_float_precision` got, and the same conclusion: it is output-visible
and wants its own commit. See
`.xo-backlog/xo-ppsink/issues/07-nested-formatting-context.md`.

## Two tests had quietly stopped testing what they claimed

Found while doing this. `tostr-fallback` and `tag_ostream-fallback-value` each
used a `double` as "a type with no Prettifier", and both carried a comment
saying so — but `Prettifier<double>` landed 2026-08-09, so neither had reached
the fallback since. They passed, because the output is identical either way,
which is exactly why nobody noticed.

Both now use a local struct with an `operator<<` and no `Prettifier<>`, with a
`static_assert(!has_prettifier<...>)` so the test cannot silently stop covering
its case again. The double assertions were kept as their own test cases, since
"the bridge still produces the same bytes" is worth pinning on its own.

**Generalisable:** a test named for a *fallback* should assert that it is on the
fallback. As `Prettifier<>` coverage grows, any test that reaches it by picking
a "type that happens not to have one" will rot the same way.

## Verification

```bash
xo-build -q --configure --with-utests --with-examples --build --install xo-ppsink
./xo-ppsink/.build/utest/utest.ppsink          # 320 assertions, 97 test cases

# the real check: no rendering anywhere in the tree moved
SUBS=$(xo-build --list | tr ' ' '\n' | grep '^xo-' \
       | while read s; do [ -f "$s/CMakeLists.txt" ] && echo "$s"; done)
xo-build -q -k --configure --with-utests --with-examples --build --install $SUBS
xo-build -q -k --utest $SUBS                   # 34 ok / 26 no-tests / 1 failed (xo-jit, pre-existing)

nix-build ci.nix -A xo-ppsink --no-out-link --check
```

The tree-wide sweep is the evidence for output-neutrality, not the ppsink
suite: the pinned expectations in xo-object2, xo-procedure2 and xo-expression2
contain integers and bools (`:id N`, `:complete 1`) and would have moved.

Mutation-checked three ways: `buf[24]`→`buf[8]` (fails the 64-bit extremes),
`1`/`0`→`true`/`false` (fails the bool case), dropping the char exclusions
(fails to compile).

**Files:**
- `xo-ppsink/include/xo/ppsink/Prettifier.hpp` — `pp_number_integral`,
  `Prettifier<T>` partial specialization, `Prettifier<bool>`
- `xo-ppsink/utest/Prettifier.test.cpp` — the three pinned cases
- `xo-ppsink/utest/{tostr,tag_ostream}.test.cpp` — the two repaired fallback tests

## Notes

This narrows what reaches pretty()'s `operator<<` branch, which is the same
direction as `.xo-backlog/milestones/ostream-containment.md`: fewer types on
the fallback means fewer headers needing `<ostream>`. It does not close any of
that milestone's 47 files — those are headers *declaring* inserters or
*including* `_ostream.hpp`, which is a different question from which types the
sink can render natively.
