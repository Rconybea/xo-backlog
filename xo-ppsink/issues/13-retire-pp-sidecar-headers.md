# 13 — retire the `_pp.hpp` sidecars: a Prettifier lives with its type

Status: open
Type: refactor
Milestone: ostream-containment
Progress: ls xo-*/include/xo/*/*_pp.hpp xo-*/include/xo/*/*/*_pp.hpp 2>/dev/null | wc -l

RC, 2026-08-16: *"we will not have `_pp.hpp` Prettifiers. They will all wind up
living with their respective types."*

## Why this is a correctness rule, not a filing preference

`xo::pp::pretty()` branches on `has_prettifier<T>` **at the point of
instantiation** (`xo-ppsink/include/xo/ppsink/pretty.hpp:56-71`). So a
`Prettifier<T>` that a translation unit can choose not to see gives two
different bodies for the same vague-linkage symbol `pretty<T>`. That is an ODR
violation. No compile error, no link error, no warning — the linker keeps one
body and every call site in the binary gets it.

Demonstrated on `TypeDescr` before it was fixed: two TUs, one including the
sidecar and one not, rendering the same `Reflect::require<double>()`.

```
link a.o b.o →  <TypeDescr ESC[33m:id ...          (Prettifier body)
link b.o a.o →  <TypeDescr ESC[38;5;245m:id ...    (operator<< body)
```

Both TUs printed the same thing within each binary, and **which thing changed
with link order**. There the divergence was tag colour. In
`.xo-backlog/xo-refcnt/issues/03` the same mechanism made the two bodies call
each other, and `utest.expression` died 54 000 frames deep.

## The hazard needs a competing path, which is why the count must reach zero

A sidecar is only dangerous when the type has *another* rendering path that a TU
without the sidecar can still compile against — otherwise the missing case is a
compile error and the ODR question never arises. So the individual headers are
not equally urgent:

```bash
for h in $(ls xo-*/include/xo/*/*_pp.hpp xo-*/include/xo/*/*/*_pp.hpp 2>/dev/null); do
  t=$(sed -n 's/.*struct Prettifier<\([^>]*\).*/\1/p' $h | head -1)
  echo "== $h  ($t)"
  # does the type ALSO have an inserter a TU could reach without this header?
  grep -rn 'operator<< *( *std::ostream' --include=*.hpp $(dirname $h)/.. 2>/dev/null \
    | grep -v '/\.build/' | grep -v "$(basename $h)"
done
```

Known as of 2026-08-17, and worth re-deriving rather than trusting:

- **`ratio_pp.hpp`** — conditional hazard. The inserter is in a separate
  `ratio_iostream.hpp`, so only a TU including that but not the sidecar
  diverges. Fewest includers of the three, so cheapest to move.
- **`span_pp.hpp`, `Refcounted_pp.hpp`** — no competing path found, so these are
  a straight relocation. `Refcounted_pp.hpp` is safe *because of*
  `.xo-backlog/xo-refcnt/issues/02` (inserters moved out of `Refcounted.hpp`,
  `operator bool` made explicit). Undo either and it becomes unsafe again, which
  is exactly why "relocate it anyway" beats "reason about it each time".

Both headers that carried a **live** hazard are already retired:

- `TypeDescr_pp.hpp` (`e2b8978c`) — `Prettifier<TypeDescrBase>` now at
  `xo-reflect/include/xo/reflect/TypeDescr.hpp:560`
- `typeseq_pp.hpp` (`d19d17c3`) — its `operator<<` shipped in `typeseq.hpp:130`,
  i.e. with the type, always visible. See also
  `.xo-backlog/xo-reflectutil/issues/01-typeseq-ostream.md`, which carries the
  same milestone and covers the inserter that made it hazardous.

Same shape as `Prettifier<gp<Object>>` in xo-alloc's `Object.hpp`, and the
`Displayable` Prettifier in `xo-refcnt/include/xo/refcnt/Displayable.hpp`.

**The three that remain are not urgent, and that is the argument for doing them
now rather than later.** None can bite today; each becomes able to the moment
someone adds an inserter for its type. Relocation is minutes per header, and it
converts the rule from something a reviewer must remember into something `ls`
answers.

## What "lives with its type" costs

The type's own header gains `#include <xo/ppsink/pretty.hpp>`. For the three
remaining that is not a new subsystem edge — each already reaches xo-ppsink.
Check before assuming:

```bash
xo-deps --why=<sub>:xo-ppsink
```

## Done when

- `Progress:` returns 0
- each relocated Prettifier renders byte-identically to before — pin the
  rendering first, per the phase-C discipline in
  `.xo-backlog/tostr-arena/spec.md`; this refactor changes no output
- `xo-build --sweep` reads
  `62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`

## Progress 2026-08-17 — two retired, one is a real decision

`span_pp.hpp` and `Refcounted_pp.hpp` are gone; their Prettifiers now sit in
`span.hpp` and `Refcounted.hpp`. Build clean, `xo-build --sweep` at
`62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`.

Neither changed rendered output, for different reasons worth distinguishing:

- **`Refcounted_pp.hpp`** — since `.xo-backlog/xo-refcnt/issues/02` a TU that
  renders an `rp<T>` without it *fails to compile*, so every such site already
  included it. Relocation is provably output-neutral, and the ten includes it
  had become redundant and were removed.
- **`span_pp.hpp`** — a TU could previously get the flat render. See below.

### CORRECTED: `span_pp.hpp` was a live hazard, and this ticket said it was not

The first version of this ticket listed span under *"no competing path found, so
these are a straight relocation"*. Wrong. `span_ppdetail.hpp:28-30` declares

```cpp
        inline std::ostream &
        operator<<(std::ostream & os,
                   const span<CharT> & x) {
```

so a TU including `span_ppdetail.hpp` but not the sidecar took pretty()'s
operator<< branch while a TU including the sidecar took the Prettifier branch —
two bodies for `pretty<span<CharT>>`, decided by the linker.

The detection grep required `std::ostream` and `span<` **on one line**. xo's
dominant style splits the signature across two, so it matched nothing.

**This is the same failure `.xo-backlog/milestones/ostream-containment.md`
already records**, where the milestone's own first query undercounted by 6x for
exactly this reason and its correction says: *"match the parameter
(`operator<<(std::ostream`) rather than the return type, since the parameter
cannot be split from the function name."* The advice was written down, in the
milestone this ticket belongs to, and was not followed — the grep here anchored
on the **argument type** instead, which splits just as easily. Anchor on
`operator<< *( *std::ostream` and nothing else.

### `ratio_pp.hpp` stays, pending RC

It is the last one, and unlike the other three it costs something real.
`xo-ratio`'s library declares only `xo_reflectutil` and `xo_flatstring`
(`xo-ratio/CMakeLists.txt:34-35`); `xo_ppsink` appears **only** in
`xo-ratio/utest/CMakeLists.txt:23`. So moving `Prettifier<ratio<Int>>` into
`ratio.hpp` puts xo-ppsink on the xo-ratio library and thence on its consumers:

```bash
xo-deps --users-of=xo-ratio --format=names -q     # 20 subsystems
```

`ratio_pp.hpp`'s own header comment argues against precisely this, and it is not
a stale argument.

Note `xo-deps --why=xo-ratio:xo-ppsink` returns **rc=0**, which reads as "the
edge already exists, so relocation is free". It is the utest edge —
`subsystem-edges` records an edge for any target — the overstatement recorded in
`.xo-backlog/tostr-arena/issues/03`. Read the CMakeLists, not the query.

The mitigating fact: `ratio_pp.hpp` and `ratio_iostream.hpp` are documented as
byte-identical *on purpose*, for this exact reason. That makes the divergence
benign today. It is still an ODR violation, and it stays benign only while
someone keeps checking. Three ways out, all RC's call:

1. relocate and accept the edge — 20 consumers gain a header-only ppsink dep
2. delete `ratio_iostream.hpp`'s inserter, leaving no competing path — then the
   sidecar is safe by the same argument that makes `Refcounted_pp.hpp` safe
3. leave it, and write the exemption into this ticket's `Progress:` so the
   counter can still reach 0

Option 2 is the one that fits the milestone, since containment wants that
inserter contained anyway.
