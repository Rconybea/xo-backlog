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
