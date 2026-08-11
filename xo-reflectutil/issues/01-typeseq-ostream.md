# 01 — move typeseq's operator<< out of typeseq.hpp

Status: open
Type: task
Milestone: ostream-containment

Progress: grep -l 'operator<< *( *std::ostream' xo-reflectutil/include/xo/reflectutil/typeseq.hpp 2>/dev/null | wc -l

`xo::reflect::typeseq` still declares an `operator<<` in `typeseq.hpp`
(`xo-reflectutil/include/xo/reflectutil/typeseq.hpp`). It must end up either in
a `typeseq_ostream.hpp` or deleted, per
`.xo-backlog/milestones/ostream-containment.md`.

**This ticket exists so the milestone cannot be closed while the inserter is
still there.** It was deliberately left in place on 2026-08-10 with the work
half-done, and a retained-for-now comment is easy to stop seeing.

## What is already done (2026-08-10)

- **`typeseq_pp.hpp` exists**: `Prettifier<typeseq>`, rendering `sink.pp(x.seqno())`
  — output-identical to the inserter, ostream-free.
- **xo-reflectutil took its first dependency**, on xo-ppsink
  (`xo_headeronly_dependency`), decided with RC. Precedent:
  `xo-ordinaltree/CMakeLists.txt:46` already does this from a header-only
  library. No cycle: `xo-deps --deps-of=xo-ppsink` is `xo-ppsink xo-timeutil`.
- **`typeseq.hpp` now includes `<ostream>` rather than `<iostream>`**, so it no
  longer instantiates `std::ios_base::Init` in every consumer. That is most of
  the practical win and it is already banked.
- Two headers that were **freeloading `<iostream>` through typeseq.hpp** now
  include it themselves: `xo-indentlog2/src/indentlog2/LogStreambuf.cpp`,
  `xo-indentlog2/utest/PpState.test.cpp`.

## What blocks the rest

Removing the inserter was attempted and reverted, so the blocker is measured
rather than predicted. **Legacy `xo-indentlog` `xtag` renders its value through
`operator<<` and has no `Prettifier` to fall back on**, so every legacy site
logging a typeseq fails to compile without it — e.g.
`xo/indentlog/print/tag.hpp:203` instantiated from
`xo-object2/src/object2/SetupObject2.cpp`.

115 call sites pass a typeseq to an `xtag`, in 21 files across 11 subsystems:

```bash
grep -rln 'xtag([^)]*typeseq::id<[^>]*>())\|xtag([^)]*_typeseq())\|xtag([^)]*\.tseq())' \
  --include=*.cpp --include=*.hpp xo-*/ | grep -v '/\.build/'
```

**Eight of the eleven still declare an xo-indentlog dependency** — measured
2026-08-10:

```bash
for s in $(grep -rln 'xtag([^)]*typeseq::id<[^>]*>())\|xtag([^)]*_typeseq())\|xtag([^)]*\.tseq())' \
           --include=*.cpp --include=*.hpp xo-*/ | grep -v '/\.build/' | cut -d/ -f1 | sort -u); do
    printf "%-18s " "$s"
    xo-deps --why=$s:xo-indentlog -q >/dev/null 2>&1 && echo "still on xo-indentlog" || echo "ppsink-only"
done
#   still on xo-indentlog: xo-expression2 xo-gc xo-interpreter2 xo-numeric
#                          xo-object2 xo-procedure2 xo-reader2 xo-type
#   ppsink-only:           xo-alloc2 xo-facet xo-reflectutil
```

So this is **downstream of `ppsink-migration`**, whose own "done when" is *no
subsystem declares an `xo-indentlog` dependency*. When that lands, the legacy
path disappears and the blocker with it. RC's call, 2026-08-10: do the work at
the end of the indentlog→ppsink migration, but hold it as a requirement of
`ostream-containment` so it cannot be quietly skipped.

## A second requirement, easy to miss

After the legacy sites are gone, the remaining `xo::pp::xtag` sites need
`Prettifier<typeseq>` **visible at the point of instantiation**. Today most of
them get it *incidentally*: `xo-facet/include/xo/facet/FacetRegistry.hpp`
includes `typeseq_pp.hpp` (for its own `dump()`), and nearly everything in the
facet cluster includes `FacetRegistry.hpp`.

That is accidental propagation, not design. Verified the hard way: with the
inserter removed and `FacetRegistry.hpp`'s include absent, `tostr(tseq)` failed
to compile — the Prettifier was simply not in scope. Sites must not be assumed
covered because they compile today.

## Suggested approach

Do not grep for the sites; **let the compiler name them**. Twice on 2026-08-10 a
grep over this exact question was wrong — once undercounting the milestone's
inventory 6x (return type on its own line), once missing
`FacetRegistry::dump()`'s `(*p_out) << kv.first.first`, which names no type at
all. The reliable procedure:

1. delete the inserter from `typeseq.hpp` (and its `<ostream>`)
2. `xo-build -q -k --configure --with-utests --with-examples --build --install $SUBS`
3. fix what the compiler names, one of:
   - a legacy `xtag` → append `.seqno()`; output is unchanged, since the
     inserter printed `seqno()` too. Precedent:
     `xo-stringtable2/src/stringtable2/SetupStringtable2.cpp`
   - a ppsink site missing the Prettifier → add `#include <xo/reflectutil/typeseq_pp.hpp>`
   - a genuine `os << tseq` → `xo::pp::tostr(x)`, or add `typeseq_ostream.hpp`
     if a real inserter is wanted (see `xo/ppsink/quoted_ostream.hpp` for the
     shape)
4. repeat until green, then full sweep + `nix-build ci.nix -A xo-reflectutil --check`

Whether `typeseq_ostream.hpp` is created at all is open. When the one bare-stream
site was rewritten with `tostr()`, **nothing in the tree needed an inserter** —
so "delete it" is a live option and is strictly better than "relocate it" if it
holds. Decide against the swept tree, not now.

**Files:**
- `xo-reflectutil/include/xo/reflectutil/typeseq.hpp` — the inserter, with a
  comment recording why it was retained
- `xo-reflectutil/include/xo/reflectutil/typeseq_pp.hpp` — the intended path
- `xo-facet/include/xo/facet/FacetRegistry.hpp` — `dump()`, the one site that
  streamed a typeseq directly; now uses `xo::pp::tostr()`

**Done when:**
- `typeseq.hpp` declares no `operator<<` and includes no `<ostream>`
- the `Progress:` query returns 0
- every remaining typeseq rendering goes through `Prettifier<typeseq>`, with
  `typeseq_pp.hpp` included where it is needed rather than arriving via
  `FacetRegistry.hpp`
