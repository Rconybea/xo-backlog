# 01 — xo-object2 free of xo-indentlog

Status: diagnosed
Type: refactor
Milestone: ppsink-migration
Progress: grep -rl '#include <xo/indentlog/' --include=*.hpp --include=*.cpp xo-printable2/include xo-printable2/src xo-stringtable2/include xo-stringtable2/src xo-object2/include xo-object2/src 2>/dev/null | grep -v '/\.build/' | wc -l

The goal is one command returning **rc=1**:

```bash
xo-deps --why=xo-object2:xo-indentlog     # today: rc=0, prints "xo-object2 -> xo-indentlog"
```

and three lines gone from `xo-cmake/etc/xo/subsystem-edges` (lines 40-42 as of
2026-08-12):

```
xo-indentlog xo-object2
xo-indentlog xo-printable2
xo-indentlog xo-stringtable2
```

This is **not** the same as `.xo-backlog/xo-printable2/issues/01`'s milestone,
which is `xo-deps --why=xo-printable2:xo-indentlog` returning rc=1. That one is
a **prerequisite** of this one, not equivalent to it.

## The three contributions, enumerated

```bash
DEPS=$(xo-deps --deps-of=xo-object2 --format=names -q | tr ' ' '\n' | grep '^xo-' | grep -v '^xo-object2$')
for d in $DEPS; do xo-deps --why=$d:xo-indentlog -q >/dev/null 2>&1 && echo "$d reaches xo-indentlog"; done
```

reports `xo-printable2` and `xo-stringtable2`, and xo-object2 additionally has
its own direct includes. So **all three must clear** before the edge can go.

`xo-stringtable2` is the one an earlier reading of this work missed entirely:
the phase-D analysis in `xo-printable2/issues/01` concluded "it doesn't affect
the milestone", which was true of *that* milestone and irrelevant to this one.

## Why the raw include count overstates the work

```bash
for s in xo-printable2 xo-stringtable2 xo-object2; do
  grep -rho '<xo/indentlog/[a-z/_]*\.hpp>' --include=*.hpp --include=*.cpp $s/include $s/src
done | sort | uniq -c | sort -rn
```

Two headers are **`pretty_deprecated` scaffolding and retire with it** at phase
E of `xo-printable2/issues/01`, needing no work here:

- `print/ppindentinfo.hpp` — the `pretty_deprecated(const ppindentinfo&)`
  declarations
- `print/pretty.hpp` — the bodies

Everything else is legacy **logging** and **exception-message construction**,
which has nothing to do with the printable facet and is the actual work below.

Counting them apart matters: a first pass at scoping phase D reported "71
production files across the cluster still include `<xo/indentlog/...>`, that's
the real work-list". About half of those retire automatically. Kept here per
rule 6 because the mistake is easy to repeat — the include census does not
distinguish *what is included for*.

## The work, per subsystem

### xo-printable2 — prerequisite, tracked elsewhere

All three files are **generated from the IDL**, so none is hand-edited:

```bash
grep -rl '#include <xo/indentlog/' --include=*.hpp --include=*.cpp xo-printable2/include xo-printable2/src
#   detail/APrintable.hpp          <- ppindentinfo.hpp, generated
#   detail/IPrintable_Xfer.hpp     <- ppindentinfo.hpp, generated
#   detail/ppdetail_Printable.hpp  <- the legacy->facet bridge
```

Done by phase E of `.xo-backlog/xo-printable2/issues/01-aprintable-pretty-ppsink.md`:
drop `ppindentinfo` from the IDL's `types:` and `includes:`, regenerate, delete
`ppdetail_Printable.hpp`. **Nothing to do here.**

### xo-stringtable2 — 3 sites

Of its 4 legacy includes, 2 are phase-E scaffolding (`DString.hpp`
ppindentinfo, `DString.cpp` pretty.hpp). Remaining:

| site | what | move to |
|---|---|---|
| `src/stringtable2/DUniqueString.cpp:63` | `scope log(XO_DEBUG(false))` — disabled, but see below | `xo::pp::scope` |
| `src/stringtable2/SetupStringtable2.cpp:25` | live `scope log(XO_DEBUG(true))` | `xo::pp::scope` |
| `src/stringtable2/SetupStringtable2.cpp:46` | live `scope log(XO_DEBUG(true))` | `xo::pp::scope` |

**Corrected 2026-08-12 (rule 6).** This table first said the `DUniqueString`
scope could be "deleted outright" because `XO_DEBUG(false)` disables it. Wrong,
and plausibly so: the scope emits nothing, but `log` is *referenced* twelve
lines later by a live `log && log(xtag(...))`, so deleting the declaration
breaks the build. Disabled is not unused. Converted like the others.

### xo-object2 — 8 sites

Of its 17 legacy includes, 13 are phase-E scaffolding (7 `ppindentinfo.hpp`
headers, 6 `pretty.hpp` bodies). Remaining, in three kinds:

**Live logging** → `xo::pp::scope` + `XO_DEBUG2_`:
- `src/object2/DList.cpp:131,144`
- `src/object2/SetupObject2.cpp:39` and its `log && log(xtag(...))` block

**Vestigial includes** — the file includes `scope.hpp` and uses no
`scope`/`XO_DEBUG`/`XO_SCOPE`; delete the line and nothing else:
- `src/object2/DBoolean.cpp`
- `src/object2/DInteger.cpp`

Found with:

```bash
for f in $(grep -rl '#include <xo/indentlog/scope.hpp>' --include=*.cpp --include=*.hpp xo-*/ | grep -v '/\.build/'); do
  grep -q 'scope log\|XO_DEBUG\|XO_SCOPE' $f || echo "vestigial: $f"
done
```

which also reports three outside this ticket's scope —
`xo-object2/utest/Printable.test.cpp`,
`xo-reader2/src/reader2/DExpectQLiteralSsm.cpp`,
`xo-reader2/src/reader2/DExpectFormalArglistSsm.cpp` — free deletions wherever
they are convenient.

**Exception messages**, `throw std::runtime_error(tostr(..., xtag(...)))` →
`xo::pp::tostr` / `xo::pp::xtag` (both exist: `xo-ppsink/include/xo/ppsink/tostr.hpp`,
`.../tag.hpp`):
- `src/object2/DArray.cpp:68`
- `src/object2/DList.cpp:118`
- `src/object2/GCObjectConversion_DInteger.cpp:31`
- `src/object2/GCObjectConversion_DFloat.cpp:31`

## Done 2026-08-12 — all 11 sites

Converted; `xo-build --sweep` →
`62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`;
`nix-build ci.nix -A xo-object2 -A xo-stringtable2 --no-out-link` green. The
`Progress:` count went **24 → 19**, and all 19 remaining are phase-E
scaffolding (`ppindentinfo.hpp` declarations, `print/pretty.hpp` bodies).

Three things worth carrying forward:

**Two idioms, not one.** Where the legacy include disappears entirely, the
namespace-scope using-declaration works — copied from
`xo-arena/src/arena/mmap_util.cpp:16-20`:

```cpp
namespace xo {
    /* the ppsink logging vocabulary, for use below */
    using xo::pp::scope;
    using xo::pp::xtag;
```

Where legacy `print/pretty.hpp` must **stay** until phase E (`DList.cpp`,
`DArray.cpp` — their `pretty_deprecated` bodies need it), `xo::scope` /
`xo::xtag` are still visible and the using-declaration is ambiguous. Those
sites are **qualified** at the call instead, with a comment saying phase E can
unqualify them.

**ADL still bites.** `xo::pp::xtag("DString.tseq", typeseq::id<DString>())` in
`SetupStringtable2.cpp` was ambiguous even with the using-declaration in scope,
because typeseq's associated namespace reaches a legacy overload and **a
using-declaration cannot suppress ADL**. Qualified at the call. Same trap
already recorded for `gp<Object>` in `.xo-backlog/xo-alloc/issues/01`.

That site also got simpler: it passed `typeseq::id<DString>().seqno()` as a
workaround for legacy xtag rendering via `operator<<`, which typeseq has none
of. On ppsink it renders through `Prettifier<typeseq>`
(`xo/reflectutil/typeseq_pp.hpp`), which emits the bare seqno — so the
`.seqno()` call is gone and the output is unchanged.

**The exception messages were unverified, and now one is pinned.**
`DArray::at` and `DList::at` build their messages with `tostr`/`xtag` and had
**no coverage at all**:

```bash
grep -rn 'out-of-range\|REQUIRE_THROWS\|CHECK_THROWS' --include=*.test.cpp xo-object2/utest
```

returned nothing. `DArray::at`'s message was compared across both vocabularies
by building each and diffing: **byte-identical, ANSI colour escapes included**
(legacy gated colour on `tag_config::tag_color_enabled`, default true; ppsink's
gate also defaults on). Pinned as `DArray-at-out-of-range` in
`xo-object2/utest/DArray.test.cpp`.

`DList::at`'s message could **not** be observed, because that function SIGABRTs
before reaching its own `throw`. Pre-existing, unrelated to this work, filed as
`.xo-backlog/xo-object2/issues/02-dlist-at-aborts-instead-of-throwing.md`.

## Ordering

**The stringtable2 and object2 work does not depend on phase E.** None of the
11 sites above touches `pretty_deprecated`; only the three *edge deletions* have
to wait for it. Doing them first means phase E ends by deleting three lines
rather than opening another round of conversions.

Suggested order:

1. the 11 logging/exception sites (this ticket), any time
2. phase E of `xo-printable2/issues/01`
3. delete the three `subsystem-edges` lines; confirm with
   `xo-deps --why=xo-object2:xo-indentlog` returning rc=1

Step 3 is where breakage surfaces, so run `xo-build --sweep` and
`nix-build ci.nix -A xo-object2 --no-out-link` after it — a removed edge changes
the installed package config, which only the nix build exercises as a real
consumer.

## What the `Progress:` count does and does not mean

It counts files under `xo-printable2`, `xo-stringtable2` and `xo-object2`
(`include/` + `src/`, production only) that include any `<xo/indentlog/...>`
header — 24 when this ticket was written. It falls monotonically and reaches
**0** exactly when all three subsystems are clean, including the phase-E
scaffolding, which is why it is a fair progress measure for the *goal* rather
than for this ticket's 11 sites alone.

It deliberately does **not** count test files. Those are a separate decision
recorded in `xo-printable2/issues/01` (RC, 2026-08-12): the legacy-protocol
tests get deleted, not converted.

## Not in scope

The other cluster subsystems — xo-procedure2, xo-expression2, xo-interpreter2,
xo-reader2 — have the same two kinds of legacy use and are **not** required for
`xo-object2 -> xo-indentlog` to go away, since nothing xo-object2 depends on
runs through them. Re-derive rather than assume, with the `--deps-of` loop at
the top of this ticket; xo-object2's upstream set is small and stable, but the
label "the cluster" is exactly the kind of group name rule 2 warns about.
