# 03 — Nested struct/vector layout: identical siblings render differently, and tags split name from value

Status: resolved (2026-08-07)
Type: bug
Milestone: ppsink-migration

## Resolution

One line, in `PpState::end` (`xo-indentlog2/src/indentlog2/PpState.cpp`):

```cpp
uint32_t tk_viz_z = scan_viz_total_ - print_viz_total_;   /* was */
uint32_t tk_viz_z = scan_viz_total_ - begin_viz_total;    /* now */
```

A group's width was measured as *scanned-so-far minus **printed**-so-far*.
`print_viz_total_` says nothing about where the group started -- and printing is
deliberately deferred until a group's fate is known, so it lags arbitrarily far
behind. Every group's measured width therefore included content scanned **before
it opened**; the deeper the nesting, the more foreign text was counted, so inner
groups reported far too wide and broke when they would have fit.

`begin()` now snapshots the scan totals into the token's size fields, and `end()`
subtracts that snapshot. Borrowing those fields is safe: `size_established()`
keys off the `k_size_established` flag (set only by `establish_size()`), and
`check_print_ready()` stops at the first token whose size is not established --
so nothing reads them as a *size* before `end()` overwrites them.

### Both symptoms were the same defect

Symptom 2 (a tag splitting `:name` from its value) needed no separate decision.
With the fit computation corrected, `:scanned_n 1` stays together on its own, so
`tag.hpp`'s `split(1, tag_config::value_offset)` is fine as designed -- it was
firing on bad arithmetic, not bad policy.

### Corrections to the analysis below

- **Not a collision with legacy ppdetail printing.** The repro was already
  ppsink-only, and `ldd` confirmed no legacy indentlog linked. The parallel-record
  test built to check that hypothesis disproved it -- and is now the regression
  test.
- The decisive clue was measurement, not code reading: at margin 80 the longest
  line produced was **34**. Something with 80 columns available using 34 is not
  making a marginal call, it is measuring the wrong thing.

### Verification

`xo-indentlog2/utest/group_fit.test.cpp` (new, 5 cases) covers: a fitting inner
group is not broken; identical siblings render identically; output narrows as the
margin narrows; tags keep name with value. **Reverting the one-line fix turns 4 of
the 5 red**, so the test bites.

Expectations updated because output improved:
- `xo-indentlog2/utest/pretty_struct.test.cpp` -- `pretty_struct-nested-accumulates-indent`
  margin 14 -> 12 (the inner correctly fits at 14 now, so the test no longer
  demonstrated its point)
- `xo-alloc/utest/{GcStatistics,ObjectStatistics}.test.cpp` -- the four pretty
  expectations, re-measured; they now sit close to the legacy `toppstr2` layout

---

## Original report

Found while migrating xo-alloc to ppsink (2026-08-07). Reproducible with no xo-alloc
involvement — this is a ppsink/xo-indentlog2 line-breaking defect.

## Repro

```cpp
#include <xo/ppsink/pretty_struct.hpp>
#include <xo/ppsink/PrettyVector.hpp>
#include "print/PrettySink.hpp"
#include <xo/arena/ArenaConfig.hpp>
#include <iostream>

struct Inner { int a=0, b=0; };
namespace xo::pp {
    template<> struct Prettifier<Inner> {
        static void print(PpSink& s, const Inner& v) {
            s.pretty_struct("Inner", field("a", v.a), field("b", v.b));
        }
    };
}

template <typename T>
std::string R(std::uint32_t margin, const T& x) {
    static int n = 0;
    xo::mm::ArenaConfig c { .name_ = "fit." + std::to_string(++n), .size_ = 64*1024 };
    xo::pp::PrettySink p(xo::pp::PpConfig()
                             .with_logbuf_config(c)
                             .with_soft_right_margin(margin), nullptr);
    p.pp(x);
    return std::string(p.output());
}

int main() {
    std::vector<Inner> v { {1,2}, {3,4} };
    for (auto m : {100u, 40u, 24u})
        std::cout << "--- margin " << m << " ---\n" << R(m, v) << "\n";
}
```

Observed:

```
--- margin 100 ---
[<Inner :a 1 :b 2>,<Inner :a 3 :b 4>]

--- margin 40 ---
[<Inner :a 1 :b 2>,<Inner
    :a 3
    :b 4>]

--- margin 24 ---
[<Inner
    :a 1
    :b 2>,
  <Inner
    :a
     3
    :b
     4>]
```

## Two distinct symptoms

### 1. Structurally identical siblings render differently

At margin 40, the first `<Inner :a 1 :b 2>` stays flat while the *identical* second
one breaks every field. At margin 24 the first breaks its fields but keeps
`:a 1` together, while the second additionally splits `:a` from `1`.

Some position-dependence is correct — a group starting further right has less room.
But the whole flat rendering at margin 40 is
`[<Inner :a 1 :b 2>,<Inner :a 3 :b 4>]`, which is **shorter than the margin**, so
nothing should have broken at all. That points at the fit computation rather than
at legitimate position-dependence.

Worth checking first: what `tk_viz_len()` holds for a `k_begin` token — whether it
measures this group's contents, or something wider (to the end of the enclosing
group, or to the end of the buffer).

### 2. A tag splits between `:name` and its value

`:a` and `3` land on separate lines at margin 24, where `:a 3` is four characters
and there is room. This is `Prettifier<tag_impl>` / `Prettifier<field_impl>`
emitting a break opportunity between name and value:

```cpp
sink.split(1, tag_config::value_offset);   /* tag.hpp:108 */
```

Whether that split should exist at all is a design question worth settling: it
lets a long value start on its own line, but for scalars it produces the mess
above. Legacy kept name and value atomic and broke only *between* fields.

## Code pointers

- `xo-indentlog2/src/indentlog2/PpState.cpp:405-407` — the `k_begin` fit test:
  ```cpp
  bool f = ((lpos + token->tk_viz_len() < config_.soft_right_margin())
            && !token->is_forced());
  ```
- `xo-indentlog2/src/indentlog2/PpState.cpp:425-440` — `k_split`: breaks iff the
  *immediately enclosing* group did not fit
- `xo-ppsink/include/xo/ppsink/tag.hpp:94-112` — `Prettifier<tag_impl>`, the
  name/value split
- `xo-ppsink/include/xo/ppsink/pretty_struct.hpp` — `Prettifier<field_impl>`, same shape
- `xo-ppsink/include/xo/ppsink/PrettyVector.hpp` — the enclosing group in the repro

## Why it matters now

**`xo-alloc/utest/{GcStatistics,ObjectStatistics}.test.cpp` currently pin this
output as expected.** They previously pinned legacy `toppstr2` output, which was
markedly better:

```
legacy:                          now:
<GcStatistics                    <GcStatistics
  :gen_v                           :gen_v
    [ <PerGenerationStatistics      [<PerGenerationStatistics :used_z 0 .. :scanned_z
        :used_z 0                       0 :survive_z 0 :promote_z 0>,
        :n_gc 0                     <PerGenerationStatistics
        ..                            :used_z
                                       0
                                      :n_gc
                                       0
                                      ..
```

Those expectations were set from measured output so the suite is green, but they
enshrine the defect. **Update them when this is fixed** — they are the regression
test for it.

**Files:**
- `xo-indentlog2/src/indentlog2/PpState.cpp` — fit computation and split handling
- `xo-indentlog2/utest/PpState.test.cpp`, `.../pretty_struct.test.cpp` — where a
  fix gets pinned (PrettySink lives here; ppsink cannot line-break on its own)
- `xo-alloc/utest/{GcStatistics,ObjectStatistics}.test.cpp` — expectations to revisit

**Done when:**
- a flat rendering that fits within the margin is not broken at all
- structurally identical siblings at the same indent render the same way
- a decision is recorded on whether tags may split between name and value
- xo-alloc's pretty expectations are updated to the improved output

## Notes

Measure, don't predict. Set every expectation from observed output — two separate
incidents on this migration came from predicting layout, and Catch2 additionally
indents both sides of a multi-line string comparison by 2 when displaying it,
which has already caused one phantom bug hunt.
