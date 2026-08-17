# 12 — inventory: which types actually reach pretty()'s operator<< fallback

Status: open
Type: analysis
Milestone: ostream-containment

Measured 2026-08-16. **776 fallback instantiations across 172 distinct types.**
Raw counts in `12-operator-fallback-inventory.types.txt` beside this file,
`count  type`, sorted descending.

No `Progress:` line: regenerating this needs a full instrumented tree build (see
Method), which is far too expensive to run per `xo-sdlc --tickets --progress`.
It is a snapshot, and it says so.

## Why this could not be done by grep

`pretty()` (`xo-ppsink/include/xo/ppsink/pretty.hpp`) picks the fallback when
`has_prettifier<T>` fails and `T` is not string-convertible. That is a property
of `T` **and** of what the translation unit included, evaluated per
instantiation — not a property of any source line. Grepping for `operator<<`
over-reports (the type may also have a `Prettifier<>`) and under-reports
(inherited and templated inserters do not match, and it cannot see which types
are actually *passed* to `pp`/`tostr`/`xtag`).

So the compiler had to enumerate it.

## Method, including two attempts that did not work

Prerequisite: `45fd03bc` made the fallback opt-in per TU — `PpSinkInserter`
carries a `std::streambuf *`, and `operator<<(PpSinkInserter&, const T&)` moved
to `pretty_ostream.hpp`, so branch 3 resolves by ADL only where that header was
included. Before that, nothing could be measured: `<ostream>` arrives from
`<sstream>`, from `FlatSink.hpp`, and from catch2, so removing one include
disabled nothing.

**Attempt 1 — hard `static_assert`.** Yields one hit and stops. `make -k`
continues only with *independent* targets, and the first hit was in xo-refcnt
(`src/Displayable.cpp:11`, `tostr0(*this)` on `xo::ref::Displayable`), which
almost everything depends on. Build died at 31% having reported one type.

**Attempt 2 — `[[deprecated]]` probe, placed after the streaming branch.**
Reported zero. The probe made `pretty.hpp` include `pretty_ostream.hpp`, so
`pp_streamable<T>` became true for every `T` and the streaming branch caught
them all; the probe branch was dead code. 591 files rebuilt and said nothing,
which is the most dangerous shape a measurement can take.

**Attempt 3 — what worked.** Probe *inside* the streaming branch:

```cpp
namespace detail {
    template <typename T>
    [[deprecated("PPSINK-FALLBACK")]]
    inline void pp_fallback_probe(PpSink & sink, const T & x) {
        auto ins = sink.stream_open(1);
        ins << x;
    }
}
/* ... in pretty(): */
} else if constexpr (pp_streamable<T>) {
    detail::pp_fallback_probe(sink, x);
```

plus `#include "pretty_ostream.hpp"` in `pretty.hpp` so the branch is reachable
everywhere.

**`-Werror` has to come off, and `CMAKE_CXX_FLAGS` cannot do it.** Subsystems
append `-Werror` *after* `CMAKE_CXX_FLAGS`, so `-DCMAKE_CXX_FLAGS=-Wno-error`
loses on gcc's left-to-right precedence and every warning stayed an error —
reproducing attempt 1's cascade. The knob is
`xo-cmake/cmake/xo_macros/xo_cxx.cmake:802`:

```cmake
set(XO_STANDARD_COMPILE_OPTIONS -Werror -Wall -Wextra)   # drop -Werror temporarily
```

then reinstall xo-cmake, reconfigure, and build:

```bash
cmake -S . -B .build && cmake --build .build -j 24 -- -k > inv.log 2>&1
grep 'PPSINK-FALLBACK' inv.log | grep -o 'with T = [^]]*' | sed 's/with T = //' \
  | sort | uniq -c | sort -rn
```

663 files, exit 0, 776 warnings. **Restore `pretty.hpp`, restore `-Werror`,
reinstall xo-cmake, and clear `CMAKE_CXX_FLAGS`** — all four, or the tree is
left quietly not-warning.

## What it found

| bucket | instantiations | share |
|---|---|---|
| `void*` family (`const void*` 307, `void*` 52, `void**` 3, `const void* const*` 1) | **363** | **47%** |
| xo domain types (Token, VsmInstr, Lambda, KalmanFilter\*, quantity, ratio, ..) | ~150 | 19% |
| other pointers (`std::byte*` 10, `SubsystemImpl*` 19, tree/GC node pointers) | ~70 | 9% |
| enums (`tokentype` 14, `exprstatetype` 6, ~20 types in all) | ~70 | 9% |
| std / third-party (`chrono::time_point` 11, Eigen, `llvm::TypeSize`, function pointers) | ~40 | 5% |
| `char` / `signed char` / `unsigned char` — deliberate, excluded by `pp_number_integral` | 13 | 2% |
| test-only types | ~10 | 1% |

Three things worth acting on separately:

**`const void*` alone is 40% of the tree's fallback traffic.** Structureless,
ubiquitous, and currently doing `stream_open` -> temporary `std::ostream` ->
`operator<<` to print a hex address. This is `09-scalar-prettifiers` again, one
type later: that ticket added `Prettifier` for every integer width after
`std::uint32_t` was found falling through. A `Prettifier` for the pointer family
would remove nearly half this inventory in one change, and it is output-visible,
so it wants its own commit and a pinned rendering first.

**The ~20 enum types all compile**, which means each already has a hand-written
`operator<<` rendering names. That is exactly the population the end state
targets — xo types whose inserter *is* their renderer — and converting them to
`Prettifier` (generating the inserter from it) is the pattern. Output-visible
across ~70 sites; needs the renderings pinned before touching.

**The genuine tail is small** — Eigen, LLVM, `chrono`, function pointers, ~40
instantiations. That supports keeping the fallback with a per-type allowlist
rather than deleting branch 3.

## Caveats on the number

- **Instantiation-based**, so it counts types this build configuration actually
  prints. A type that could fall back but is never printed does not appear —
  correct for "what degrades today", wrong for "what could".
- **This build**: the umbrella `.build` with utests and examples. Anything gated
  off (a disabled example, a subsystem not in `.build`) is invisible.
- Counts are *instantiations*, not call sites: one `T` printed from ten TUs
  counts ten times. Useful as a frequency proxy, not as a work estimate.
- The 3.2 MB raw log was not kept. It carries per-call-site attribution (each
  warning has the instantiation backtrace naming file and line), so regenerate
  if you want "who prints this", not just "what".

## Read this list in TWO buckets, not one

Added 2026-08-16 after the first triage pass. A type appearing here means one of
two quite different things, and the fix differs:

| symptom | fix | getting it wrong costs |
|---|---|---|
| **no `Prettifier<T>` exists** (`Displayable`, `void*`, enums, domain types) | write one — or include `pretty_ostream.hpp` deliberately, as a stopgap | nothing; the include is honest |
| **a `Prettifier<T>` exists but is not reachable from this TU** (`rp<T>`, `Expression`, anything with a `_pp` header) | include the `_pp` header that supplies it | **silent loss of structure** |

The second is the dangerous one, and this list cannot distinguish it — both
appear as "T reached the fallback".

**Worked example.** `xo-interpreter/src/interpreter/ExpressionBoxed.cpp` failed
under the new guard. Its `pretty()` is already converted and its body is
`sink.pp(contents_)` where `contents_` is `rp<Expression>`. Adding
`pretty_ostream.hpp` compiled — and rendered the boxed expression flattened
through an ostream, discarding exactly the structure the comment three lines
above claims to preserve (*"Going through the sink rather than an ostream means
the boxed value now nests instead of flattening"*). The right include was
`<xo/expression/pretty_expression.hpp>`, which supplies `Prettifier<Expression>`
and pulls in `Refcounted_pp.hpp` for the `rp<T>` forwarder.

`Refcounted_pp.hpp`'s own header comment predicted this in general terms:
*"WITHOUT this header, rp<T> ... flattens the pointee through an ostream and
discards all group structure. That is silent: the output is still readable, it
just never wraps."*

So before adding `pretty_ostream.hpp` anywhere, check whether a `Prettifier` for
the type already exists:

```bash
grep -rn 'struct Prettifier<' --include=*.hpp xo-*/ | grep -v '/\.build/'
```

NB that query must be read for concept-constrained partial specializations too —
`template <pp_number_integral T> struct Prettifier<T>` is spelled
`struct Prettifier<T>` and matches no type name. Filtering that output by type
name is how this session twice concluded a Prettifier did not exist when it did.

## Status of the number itself, 2026-08-16 (later)

**The 776 / 172 figures are now stale, in a known direction.** They were measured
before `Prettifier<void *>` / `Prettifier<const void *>` landed
(`xo-ppsink/include/xo/ppsink/Prettifier.hpp`, with the null case pinned in
`utest/Prettifier.test.cpp`). Those covered 359 of the 363 pointer
instantiations, and `xo-facet/include/xo/facet/FacetRegistry.hpp` alone was 303
of them, so both should be gone. Re-running is much cheaper now that the recipe
above is written down, and the remaining number is the one that should guide the
next round.

## Where the tree ended up

`xo-build -q --sweep` is green — `62 attempted: 34 ok, 28 with no tests,
0 failed, 0 skipped` — with the fallback opt-in per TU tree-wide. Fifteen files
include `pretty_ostream.hpp`; ten are ppsink's and indentlog2's own tests, one is
`pretty.hpp`, one is `webutil_ostream.hpp` (a bridge, correct by construction).
The three worth review:

- `xo-refcnt/src/Displayable.cpp` — genuine stopgap; `.xo-backlog/xo-refcnt/issues/01`
  is the real fix (convert `display()` to `pretty()`)
- ~~`xo-reactor/include/xo/reactor/Sink.hpp`~~ — **resolved 2026-08-16**: the
  include was vestigial.  RC dropped it; `xo-build -q --sweep` stayed green
  (`62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`), which is the
  check that matters since Sink.hpp is a public header with eleven downstream
  subsystems -- building xo-reactor alone would not have proved it.
- `xo-pyunit/src/pyunit/pyunit.cpp` — unexamined

**Note the technique.** "Drop the include and see whether anything breaks" is
only a valid experiment *because* the fallback is now opt-in.  Under the old
regime an unnecessary `pretty_ostream.hpp` was indistinguishable from a
necessary one: the fallback compiled either way, since `<ostream>` arrives from
`<sstream>`, `FlatSink.hpp` and catch2 regardless.  So every remaining
`pretty_ostream.hpp` include in the tree is now cheaply falsifiable, and some
others are probably vestigial too.
