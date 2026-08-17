# 02 — move rp<T> / Borrow<T> inserters to Refcounted_ostream.hpp

Status: fixed 2026-08-16
Type: refactor
Milestone: ostream-containment
Progress: grep -c 'operator<<(std::ostream' xo-refcnt/include/xo/refcnt/Refcounted.hpp

RC's plan, 2026-08-16. `Refcounted.hpp` carries two ostream inserters:

```bash
grep -n 'operator<<(std::ostream' xo-refcnt/include/xo/refcnt/Refcounted.hpp
#   :239  intrusive_ptr<T>   (i.e. rp<T>, aliased at Refcounted.hpp:23)
#   :352  Borrow<T>
```

Both move to a new `xo-refcnt/include/xo/refcnt/Refcounted_ostream.hpp`.
Naming follows the family already in place — `Refcounted_pp.hpp`,
`Refcounted_indentlog.hpp` — and the latter's header comment is the voice to
copy: it states what the header is for and when to stop needing it.

## What this buys, beyond tidiness

It removes a **silent** failure mode, and `Refcounted_pp.hpp` already documents
that mode against itself:

> *WITHOUT this header, `rp<T>` falls through ppsink's leaf path to
> `operator<<(std::ostream&, intrusive_ptr<T> const&)` (Refcounted.hpp), which
> flattens the pointee through an ostream and discards all group structure.
> That is silent: the output is still readable, it just never wraps.*

Today a site printing an `rp<T>` gets structure or doesn't, depending on whether
something happened to include `Refcounted_pp.hpp` — and only two files in the
tree do:

```bash
grep -rln 'Refcounted_pp.hpp' --include=*.hpp --include=*.cpp xo-*/ | grep -v '/\.build/'
#   xo-expression/include/xo/expression/pretty_localenv.hpp
#   xo-expression/include/xo/expression/pretty_expression.hpp
```

The inventory measured the consequence: ~17 `intrusive_ptr<T>` instantiations
reach pretty()'s fallback — `AbstractSink` x5, `KalmanFilterStateExt` x3,
`KalmanFilterInput` x2, `TypeBlueprint` x2, `Expression` x2, plus
`KalmanFilterState`, `ReactorSource`, `Variable`
(`.xo-backlog/xo-ppsink/issues/12-operator-fallback-inventory.md`). Every one is
a structured pointee being flattened, invisibly.

**After the move, combined with `45fd03bc`** (which made pretty()'s operator<<
fallback opt-in per TU), a site with neither header is a compile error rather
than a quiet downgrade. So each one becomes an explicit choice:

| site includes | result |
|---|---|
| `Refcounted_pp.hpp` | structured — the pointee's own Prettifier |
| `Refcounted_ostream.hpp` | flat, deliberately |
| neither | does not compile |

That is the whole point: the silent middle case disappears.

## Sizing it

Do **not** try to grep for the call sites. `os << x` where `x` is an `rp<T>` is
not distinguishable by text, and the same lesson has been learned three times on
this migration — an include census is a lower bound, never a work-list. Move the
inserters, build, and let the compiler enumerate. That technique is what
produced issue 12's inventory, and it is exact.

`Refcounted.hpp` has 27 includers, so expect the first build to be noisy.

## One expected win that does NOT follow

Moving the inserters does **not** let `Refcounted.hpp` drop
`#include <xo/ppsink/tostr0.hpp>` (and with it `<sstream>`, and so
`std::ostream`). That include is also used by the aliasing-constructor error at
`Refcounted.hpp:103-107`:

```cpp
throw std::runtime_error(tostr0("attempt to use aliasing ctor with",
                                xtag("Y", reflect::type_name<Y>()),
                                xtag("T", reflect::type_name<T>())));
```

So a very widely included header keeps pulling in the stringstream machinery
either way. Making `Refcounted.hpp` genuinely ostream-free is a separate
question about that one throw, and should not be smuggled into this ticket or
assumed to come free with it.

## Displayable's inserter — a naming decision

`Displayable.hpp:20-24` has its own inline
`operator<<(std::ostream&, Displayable const&)`, which
`.xo-backlog/xo-refcnt/issues/01` also wants moved. Two defensible homes:

- fold it into `Refcounted_ostream.hpp` — one bridge header for the subsystem
- give it `Displayable_ostream.hpp` — one bridge header per source header,
  matching how `Refcounted_pp.hpp` pairs with `Refcounted.hpp`

Worth settling once, since whichever is chosen sets the pattern for the rest of
`ostream-containment`'s 125 headers. Issue 01 currently says "refcnt_ostream.hpp"
in passing; that spelling is wrong for this family either way.

## Done when

- `Progress:` returns 0 — no `operator<<(std::ostream` left in `Refcounted.hpp`
- every site that streams an `rp<T>`/`Borrow<T>` includes one of the two bridge
  headers deliberately, and the ~17 sites from issue 12 are triaged rather than
  blanket-converted: a structured pointee wants `Refcounted_pp.hpp`, not the
  ostream bridge
- `xo-build --sweep` reads
  `62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`
- `nix-build ci.nix -A <a consumer> --no-out-link` — xo-refcnt's public header
  surface changes, so the check is a consumer, not xo-refcnt

## Progress 2026-08-16 — inserters moved (`944c089c`), triage NOT done

`Refcounted.hpp` is clean: `Progress:` returns 0, and
`xo-refcnt/include/xo/refcnt/Refcounted_ostream.hpp` exists. The naming question
above was settled in favour of **one bridge header for the subsystem** —
`Displayable`'s inserter did not need a home after all, because `issues/01`
removed it rather than moving it.

The move landed, but two things the ticket assumed turned out to be false, and
together they invert its central claim.

### 1. The header as committed did not compile

It declared both inserters in `namespace xo::pp`, where `intrusive_ptr` and
`Borrow` are unqualified names that do not resolve:

```
Refcounted_ostream.hpp:12: error: 'intrusive_ptr' has not been declared
Refcounted_ostream.hpp:23: error: 'Borrow' has not been declared
```

Nothing in the tree includes the header yet, so `cmake --build .build` was green
with a header that compiles nowhere. Moved to `namespace xo::ref` and pinned by
a new `xo-refcnt/utest/Refcounted_ostream.test.cpp` (3 cases: rp, null rp, bp).

`xo::ref` is not a style preference here. ADL for `os << p` where `p` is an
`xo::ref::intrusive_ptr<T>` associates `xo::ref` and `T`'s own namespace;
**`xo::pp` is not associated**, so an inserter declared there is never found.
Contrast `pretty_ostream.hpp`, whose `operator<<` *is* correctly in `xo::pp` —
its left operand is `xo::pp::PpSinkInserter`, so `xo::pp` is associated there.
The two headers look parallel and are not.

### 2. "Neither header -> does not compile" is FALSE for rp/bp

The table above promised the silent middle case disappears. It does not.
`intrusive_ptr<T>` and `Borrow<T>` both have a **non-explicit `operator bool`**
(`xo-refcnt/include/xo/refcnt/Refcounted.hpp:134` and `:299`), so with no
inserter visible `os << p` still compiles — and prints `1`.

Measured, with a `Foo : ref::Refcount` in a third namespace and its own ostream
inserter:

| includes | `tostr0(rp<Foo>)` renders |
|---|---|
| `Refcounted_ostream.hpp` (fixed, in `xo::ref`) | `Foo(17)` |
| the header as committed (in `xo::pp`) | `1` |
| neither bridge header | `1` |

So `944c089c` is a **silent output regression** at every site that renders an
`rp<T>` without `Refcounted_pp.hpp` — the same ~17 instantiations issue 12
inventoried (`AbstractSink` x5, `KalmanFilterStateExt` x3, `KalmanFilterInput`
x2, `TypeBlueprint` x2, `Expression` x2, `KalmanFilterState`, `ReactorSource`,
`Variable`). They previously flattened; they now print `1` or `0`. The sweep
does not catch it, which means none of those renders is pinned by a test.

**This is the correction that matters for the whole `ostream-containment`
milestone.** "Move the inserters, build, and let the compiler enumerate" — this
ticket's own sizing advice, and the technique that produced issue 12's exact
inventory — is only valid for types with **no implicit conversion to something
streamable**. A smart pointer with `operator bool` degrades silently instead.
Before applying the technique to any of the remaining 125 headers, check the
type for conversion operators; `grep -n 'operator bool\|operator [A-Z]' <hdr>`
is the cheap version.

### Still open

- triage the ~17 sites: each wants `Refcounted_pp.hpp` (structured, the likely
  answer for all of them — every pointee named is struct-shaped) or the ostream
  bridge, deliberately
- until they are triaged the tree renders `1` where it used to render the
  pointee, so this is a live defect, not just unfinished tidying
- worth considering instead: move `Prettifier<intrusive_ptr<T>>` /
  `Prettifier<Borrow<T>>` **into `Refcounted.hpp`**, which already includes
  `<xo/ppsink/tostr0.hpp>` so costs nothing new, and which would make the
  structured render the default rather than an opt-in that 25 of 27 includers
  forgot. That reverses this ticket's premise and should be argued, not assumed

## RESOLVED 2026-08-16 — `explicit operator bool`, and the triage it enumerated

RC's suggestion, and it is a better answer than the "put Prettifier in
Refcounted.hpp" reversal floated above. Making both conversions explicit

```cpp
/* xo-refcnt/include/xo/refcnt/Refcounted.hpp:134, :299 */
explicit operator bool() const { return ptr_ != nullptr; }
```

restores the property the ticket wanted: a site with neither bridge header is a
compile error, because the `bool` escape hatch is what made it silent.

### Measured blast radius: one test line

`cmake --build .build -j -- -k` with the change, before any triage:

| category | count | what |
|---|---|---|
| intended catch | 11 TUs, 8 pointee types | all at `pretty_ostream.hpp:70` |
| collateral | **1** | `xo-refcnt/utest/intrusive_ptr.test.cpp:79` |

The collateral was `REQUIRE((bool)p1_brw == false)` — the explicit cast is not
enough, because catch2's decomposer captures the operand before the comparison
and then wants `Borrow<T> == bool`. Rewritten as `REQUIRE(!p1_brw)`.

**Nothing else in 62 subsystems relied on the implicit conversion.** That is the
number that makes this cheap, and it was not predictable in advance — `if (p)`,
`!p`, `p && q` and ternary conditions are all *contextual* conversions, which
`explicit` still permits. Only the non-contextual uses break, and the tree had
one.

### The 8 types, now compiler-enumerated rather than probe-inventoried

`rp<>`: `KalmanFilterInput`, `KalmanFilterState`, `KalmanFilterStateExt`,
`AbstractSink`, `ReactorSource`, `TypeBlueprint`, `Webserver`.
`bp<>`: `TypeBlueprint`.

That is issue 12's list reproduced from a build rather than from a
`[[deprecated]]` probe — `Expression` and `Variable` do not appear because
xo-expression's `pretty_expression.hpp` already includes `Refcounted_pp.hpp`,
which is the correct outcome, not a miss.

Every one was triaged to **`Refcounted_pp.hpp`** (structured), none to the
ostream bridge — as predicted, since every pointee named is struct-shaped:

```
xo-expression/src/expression/typeinf/type_unifier.cpp
xo-kalmanfilter/src/kalmanfilter/{KalmanFilter,KalmanFilterSpec,KalmanFilterStep}.cpp
xo-simulator/src/simulator/SourceTimestamp.cpp
xo-websock/src/websock/Webserver.cpp
xo-reactor/include/xo/reactor/{Sink,PolyAdapterSink}.hpp    <- public headers
```

So `Refcounted_ostream.hpp` currently has **zero includers**, which is the right
result for it: it exists as the paved road for a deliberate flat render, and
nothing in the tree wants one yet. Keep the utest, which is the only thing
compiling it.

### One gap this uncovered, belonging to `issues/01`

`xo-websock` did not fix with `Refcounted_pp.hpp` alone. Its forwarder does
`sink.pp(*p)`, and `Webserver` — converted by `issues/01` — has
`pretty(PpSink&)` but no `Prettifier`. A virtual member function is not what
`pretty.hpp`'s dispatch looks for, so it fell through to `operator<<`, which
`issues/01` had deleted. The error surfaces at `pretty_ostream.hpp:70` naming
`const xo::web::Webserver`, well away from the cause.

Fixed by a new `xo-refcnt/include/xo/refcnt/Displayable_pp.hpp`:

```cpp
template <typename T> requires std::derived_from<T, xo::ref::Displayable>
struct Prettifier<T> { static void print(PpSink & sink, const T & x) { x.pretty(sink); } };
```

Constrained rather than one specialization per class, so converting a class to
`Displayable` is sufficient — it does not also have to be registered.

**Generalize this.** Converting a hierarchy to `pretty(PpSink&)` is only half
the job; without a Prettifier the override is unreachable from `xo::pp::pretty`.
`xo-alloc/issues/01` got this right for `gp<Object>` and said so; this ticket is
the second instance, so it is a pattern rather than an oversight — worth stating
as a rule in whatever `ostream-containment` uses for guidance.
