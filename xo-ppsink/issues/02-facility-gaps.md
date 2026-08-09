# 02 — Legacy printing facilities with no xo-ppsink equivalent

Status: open (1 facility left)
Type: task
Milestone: ppsink-migration

Inventory of what xo-indentlog still supplies that xo-ppsink does not, so the
indentlog→ppsink migration order can be chosen with the blockers visible.

Motivation: on the xo-ratio migration, `hex_view` was discovered missing only
*after* the work had been scoped, and porting it (plus making `xo::mm::span` and
`xo::scm::span` model `std::ranges::contiguous_range`) became three commits that
were not in the plan. Cheap to avoid by knowing the gaps up front.

Originally measured 2026-08-06 (7 blocking). **Re-measured 2026-08-08 (2 left).**

"Real consumers" excludes `xo-indentlog/utest/*`, which stays on legacy and is
not a migration blocker.

## Still unported (1)

| Facility | Real consumers | In a public header? |
|---|---|---|
| `print/cond.hpp` | `xo-expression2/src/expression2/{TypeRef,DDefineExpr}.cpp`, `xo-reader2/src/reader2/DProgressSsm.cpp` | no |

**`cond`** chooses between two values rather than formatting one, so it is a
different kind of thing from the rest. Decide whether ppsink wants it *at all*
before porting — its three call sites may read better rewritten without it.
It was deliberately left out of `pretty_struct` v1. All three consumers are
inside the facet cluster, which is gated on the
`IPrintable::pretty(ppindentinfo)` question regardless, so this is not on any
critical path. Verified 2026-08-08:

```bash
xo-deps --why=xo-expression2:xo-printable2  # xo-expression2 -> xo-printable2
xo-deps --why=xo-reader2:xo-printable2      # xo-reader2 -> xo-expression2 -> xo-printable2
```

### `print/printer.hpp` was never a gap (retired 2026-08-08)

It was listed as blocking xo-process on the strength of a `grep` hit. The two
hits in `xo-process/utest/RealizationSource.test.cpp` were the `#include` and a
**commented-out** `using xo::print::printer;`. Nothing instantiated it. Both
lines deleted during the xo-process migration; no ppsink equivalent written.

Worth generalising: **before scoping a port, check the consumer actually
instantiates the facility**, not merely that its name appears. A commented-out
`using` and a dead include read exactly like a live consumer to `grep`, and here
that cost xo-process a spurious "blocked" label for two days. The reverse error
(`hex_view`, missed entirely until mid-refactor) is what this ticket exists to
prevent — but over-reporting has a cost too, in migration order chosen around a
blocker that isn't there.

## Ported since (5)

| Facility | ppsink equivalent | Residual legacy call sites |
|---|---|---|
| `print/array.hpp` + `ppdetail<std::array>` | `pretty_array.hpp` (`Prettifier<std::array<T,N>>`) | `xo-gc/utest/Collector.test.cpp` |
| `print/pair.hpp` | `pretty_pair.hpp` (`Prettifier<std::pair<T,U>>`) | **none** |
| `print/pad.hpp` | `pad.hpp` (`pad()`, `spaces()`, `Prettifier<pad_impl>`) + `pad_ostream.hpp` | **none** |
| `print/ppstr.hpp` (`toppstr2`) | **retired, not ported** — superseded by `PrettySink`, which is what actually line-breaks | none |
| `print/time.hpp` | `pp_time.hpp` value wrappers + `pp_time_ostream.hpp` | **none** |

The residual call sites above are *not* blocked — the ppsink equivalent exists;
those files simply have not been migrated yet.

### `print/time.hpp` is now dead

As of the xo-printjson migration (2026-08-08) it has **zero includers
tree-wide** outside xo-indentlog's own tests. That retires more than three
wrapper types: the header also declared

```cpp
namespace std { namespace chrono {
    inline std::ostream & operator<<(std::ostream &, xo::time::utc_nanos);
    inline std::ostream & operator<<(std::ostream &, xo::time::nanos);
}}
```

— inserters injected into `namespace std::chrono`, which is undefined behaviour
and silently governed how *every* time_point in the program printed. ppsink
replaces them with `Prettifier<utc_nanos>` / `Prettifier<nanos>`, which are
legal (they live in `xo::pp` and are found by the Prettifier machinery, not by
ADL) and preserve the same space-free formats.

**Known limit, documented in `pp_time_ostream.hpp`:** the Prettifiers cover every
ppsink path (`sink.pp(t)`, `xtag("k", t)`, `tostr(t)`, `pp_to_stream(os, t)`) but
NOT a raw `os << t`, which resolves by ADL into `std::chrono` and so gets C++20's
default — `2022-09-26 09:30:00.123456000`, which embeds a space and therefore
does not read back as a single token. Use `pp_to_stream()` or an explicit
wrapper.

### Missing from this inventory until 2026-08-09: `print/fixed.hpp`

`xo::fixed(x, prec)` -- a value wrapper rendering a double at a given
precision. Not listed above, in either the unported or the ported table, though
this ticket exists to be a complete inventory.

**Zero real consumers**, so it never blocked anything, which is presumably why
nothing surfaced it:

```bash
grep -rln 'print/fixed.hpp' xo-*/ --include=*.hpp --include=*.cpp | grep -v '/\.build/'
#   xo-indentlog/utest/fixed.test.cpp    -- its own test, excluded by this
#                                           ticket's "real consumers" rule
```

It is worth recording anyway, for two reasons.

**It is the existing precedent for float formatting control.** The
implementation saves `flags()`/`precision()`, sets its own, prints, restores --
i.e. legacy already treated ambient stream state as something to defend against
rather than use. That bears directly on
`.xo-backlog/xo-ppsink/issues/07-nested-formatting-context.md`, where the
question is whether formatting should be controlled by value wrappers or by a
nested context: `fixed` is proof the wrapper approach was already in use for
precisely this case.

**A gap with no consumers is exactly what an inventory is for.** The whole
point of this ticket is that `hex_view` was discovered missing only after work
had been scoped. A facility with no users today is the one most likely to be
missed when a consumer appears.

Whether it wants porting at all is open --
`Prettifier<double>` (added 2026-08-09) now owns the default, so a ppsink
`fixed` would be an override of that default, and issue 07 may subsume it.

## Deliberately absent, not a gap

- **`ppindentinfo.hpp`** — the legacy two-pass `upto()` fit protocol. ppsink's
  model is single-pass (`Prettifier<T>::print(PpSink&, const T&)`), so there is
  nothing to port. Those sites are ppdetail specializations to *rewrite*, and
  they are the bulk of the remaining migration.

  **Re-measured 2026-08-08** (the earlier "~293 files, concentrated in the facet
  cluster" was carried, not checked):

  ```bash
  # 298 files tree-wide, excluding xo-indentlog itself
  grep -rl "ppdetail\|ppindentinfo" xo-*/ --include=*.hpp --include=*.cpp \
    | grep -v '\.build' | grep -v '^xo-indentlog/'
  # split by whether they are downstream of xo-printable2
  xo-deps --users-of=xo-printable2 --format=names -q
  ```

  298 files: **233 inside** the printable2 blast radius (reader2 92,
  expression2 46, interpreter2 34, object2 29, procedure2 11, stringtable2 8,
  tokenizer2 7, printable2 6), **65 outside** it — and the outside group is not
  noise: xo-expression 28 and xo-reader 24 account for 52 of the 65. Those two
  carry a *separate* two-pass protocol
  (`GeneralizedExpression::pretty_print(ppindentinfo)`) and are independent of
  the facet-cluster decision. See
  `.xo-backlog/xo-expression/issues/01-ppsink-migration-pilot.md`.

  So "the remaining migration is the facet cluster" is **false** — it is the
  facet cluster (233) plus an independent expression/reader stack (52). See
  `.xo-backlog/CONVENTIONS.md` rule 2.
- **`ppdetail_atomic.hpp`** — legacy's "declare this type a leaf". ppsink's
  primary `Prettifier` template is empty, so an unspecialized type already falls
  through to the string-like leaf and then `operator<<`. No equivalent needed.
  NB every `#ifndef ppdetail_atomic` block in the tree is **dead code**:
  `ppdetail_atomic.hpp` defines that macro unconditionally. Delete rather than
  port (done so far in xo-unit, xo-reactor).

## Suggested approach

Port on demand, in migration order, rather than all at once — each port wants a
real consumer to validate its shape, and `hex_view` showed how much the call
sites constrain the API (pointer-pair vs range ctor, enum vs bool). But size the
work with this table before committing to a subsystem, so nothing surfaces
mid-migration.

**Files:**
- `xo-ppsink/include/xo/ppsink/` — where each port lands
- `xo-ppsink/utest/` + `xo-indentlog2/utest/` — flat/token-stream tests here,
  rendered line-breaking tests there (ppsink has no PrettySink)
- `xo-indentlog/include/xo/indentlog/print/` — the originals

**Done when:**
- each remaining facility either has a ppsink equivalent, or its consumers have
  been rewritten not to need it

## Notes

**Compare API *shapes*, not facility names.** The original pass matched
`print/time.hpp` to `pp_time.hpp` because both format ISO-8601, and marked it
covered — but legacy offered a *value wrapper* and ppsink only a *sink-writing
function*, so the call site could not be ported by swapping a name. Before
relying on any "ported" row above, check the ppsink version can stand where the
legacy one stood: as a value, in an ostream expression, and as a tag value.

Measure rendered output before writing test expectations; two separate incidents
on this migration came from predicting layout instead of observing it.
