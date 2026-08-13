# 11 — treat xo::pp::tostr as the fallback, not the default

Status: hypothesised
Type: refactor
Progress: grep -rl 'xo/ppsink/tostr.hpp' --include=*.hpp --include=*.cpp xo-*/ | grep -v '/\.build/' | wc -l

`xo::pp::tostr` (`xo-ppsink/include/xo/ppsink/tostr.hpp`) builds its result in a
`std::stringstream` and returns a heap `std::string`. It is currently the
default one-off-string builder across xo — **109 files** as of 2026-08-13:

```bash
grep -rl 'xo/ppsink/tostr.hpp' --include=*.hpp --include=*.cpp xo-*/ | grep -v '/\.build/' | wc -l
grep -rl 'xo/ppsink/tostr.hpp' --include=*.hpp --include=*.cpp xo-*/ | grep -v '/\.build/' | sed 's|/.*||' | sort | uniq -c | sort -rn
```

RC's proposal (2026-08-13), two parts:

1. prefer `xo::pp::toppstr` (xo-indentlog2) **where appropriate and where it
   introduces no new dependency**
2. where that does not make sense, add an **arena-backed** `tostr` — same
   flattening semantics, but writing into arena memory instead of a
   `stringstream`

with the expected side benefit of dropping some `*_ostream.hpp` includes.

The `tostr.hpp` header already anticipates part 2, which is worth quoting since
it also names the constraint:

> NB: this uses std::stringstream / std::string (heap). An *arena-backed*
> tostr (building the result into arena memory) is a separate, higher-level
> facility — it cannot live here because xo-ppsink sits below xo-arena in the
> level order.

## The two are not interchangeable — semantics first

| | `xo::pp::tostr` | `xo::pp::toppstr` |
|---|---|---|
| lives in | xo-ppsink | xo-indentlog2 |
| sink | `FlatSink` over `std::stringstream` | `PrettySink` over an arena `LogBuffer` |
| line breaking | **never** — flattens all structure | **yes**, at the configured margin |
| operator<< fallback | yes — includes `pretty_ostream.hpp` | **not supplied by the header** (see below) |

So a blanket `tostr` -> `toppstr` substitution is **not behaviour-preserving**:
anything wide enough gains newlines. For the dominant use — one-line exception
messages and labels — that is a regression, not an improvement. Any migration
has to be judged per call site, or done with a margin wide enough to suppress
breaking (in which case the arena backing, not the pretty-printing, is the
actual win — which is part 2, not part 1).

**Unverified, and worth checking early:** `toppstr.hpp` includes
`PrettySink.hpp` -> `PpSink.hpp`, but *not* `pretty_ostream.hpp`, whereas
`tostr.hpp` does include it. If that means `toppstr` fails to compile for a
type whose only rendering is `operator<<` — where `tostr` succeeds — then the
set of call sites part 1 can touch is smaller than it looks, and the difference
should be documented in both headers.

## Levelization decides most of it

```bash
xo-deps --deps-of=xo-indentlog2 --format=names -q
#   xo-arena xo-ppsink xo-randomgen xo-reflectutil xo-subsys xo-testutil xo-timeutil
```

Four current `tostr` users are **upstream of xo-indentlog2** and therefore can
never call `toppstr`, whatever the merits:

```bash
for s in xo-ppsink xo-arena xo-subsys xo-testutil; do
  xo-deps --why=xo-indentlog2:$s -q >/dev/null 2>&1 && echo "$s is upstream of xo-indentlog2"
done
```

and **15 subsystems** would acquire a *new* xo-indentlog2 dependency from a
naive swap:

```bash
for s in $(grep -rl 'xo/ppsink/tostr.hpp' --include=*.hpp --include=*.cpp xo-*/ | grep -v '/\.build/' | sed 's|/.*||' | sort -u); do
  xo-deps --why=$s:xo-indentlog2 -q >/dev/null 2>&1 || echo "$s"
done
#   xo-arena xo-distribution xo-facet xo-flatstring xo-indentlog2 xo-ppsink
#   xo-pyunit xo-ratio xo-refcnt xo-reflect xo-subsys xo-testutil xo-tokenizer
#   xo-unit xo-webutil
```

Adding a dependency to make a string is the wrong trade in most of those. **So
part 1 applies only to subsystems that already depend on xo-indentlog2** — a
much smaller set, and the `--why` loop above is how to enumerate it.

## Where an arena-backed tostr should live: xo-arena, not xo-indentlog2

The `tostr.hpp` comment says such a facility cannot live in xo-ppsink because
ppsink is below xo-arena. It does **not** follow that it belongs in
xo-indentlog2. Measured:

```bash
xo-deps --why=xo-arena:xo-ppsink     # xo-arena -> xo-ppsink   (ppsink BELOW arena)
xo-deps --why=xo-ppsink:xo-arena     # rc=1                     (ppsink does not need arena)
```

so the level order is `xo-ppsink < xo-arena < xo-indentlog2`, and **xo-arena is
the lowest level at which an arena-backed tostr can live**. Putting it there
serves every consumer above xo-arena, including xo-subsys and xo-testutil,
which xo-indentlog2 cannot reach. Putting it in xo-indentlog2 would exclude
them for no reason.

Whether xo-arena is the right *home* in a design sense — as opposed to merely
the lowest legal one — is a separate question, and this ticket does not settle
it. But the levelization argument should be made on measurement, not on the
observation that `toppstr` already happens to live in xo-indentlog2.

## The `*_ostream.hpp` claim, examined

Current includers:

```bash
for h in xo-ppsink/include/xo/ppsink/*_ostream.hpp; do
  printf "%-26s %3d\n" "$(basename $h)" \
    "$(grep -rl "xo/ppsink/$(basename $h)" --include=*.hpp --include=*.cpp xo-*/ | grep -v '/\.build/' | wc -l)"
done
```

| header | includers |
|---|---|
| `tag_ostream.hpp` | **143** |
| `quoted_ostream.hpp` | 9 |
| `pretty_ostream.hpp` | 8 |
| `pad_ostream.hpp` | 3 |
| `pp_time_ostream.hpp`, `quoted_char_ostream.hpp` | 2 each |
| `hex_ostream.hpp`, `log_level_ostream.hpp` | 1 each |

Two distinct effects, worth not conflating:

- **`pretty_ostream.hpp` (8)** is pulled in by `tostr.hpp` itself, for the
  `operator<<` fallback. A tostr variant that rendered *only* through
  `Prettifier<T>` would not need it — but would then fail for any type whose
  rendering is an `operator<<`, which is most legacy leaf types. That is a
  policy change (refuse opaque leaves), closely related to the fallback
  question in `.xo-backlog/xo-interpreter2/issues/01`, where the leaf fallback
  silently printed a pointer address.
- **`tag_ostream.hpp` (143)** is the dominant header, and it is **not** reached
  through `tostr` at all. It exists for `os << xtag(...)` inside
  `print(std::ostream&)` methods. Dropping those includes means rewriting those
  methods to build a string (via whichever tostr) or to take a `PpSink &` —
  i.e. the ostream-containment work, not this ticket. See the
  `ostream-containment` milestone.

So the side benefit is real but small for `pretty_ostream`, and the large
number belongs to a different migration. Stating it that way avoids this ticket
being read as "this removes 143 includes".

## Suggested shape

1. settle the `toppstr` + `operator<<` question above — it bounds part 1
2. enumerate the subsystems that already depend on xo-indentlog2 **and** use
   `tostr`; within those, convert only call sites where line breaking is
   wanted, or harmless
3. decide the home for an arena-backed flattening `tostr` (xo-arena is the
   lowest legal level; whether it is the right one is open) and implement it
4. leave `xo::pp::tostr` in place as the fallback for everything below
   xo-arena, and say so in its header comment — the point of this ticket is to
   make it the deliberate fallback rather than the default reached for first
