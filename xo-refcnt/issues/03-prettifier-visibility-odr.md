# 03 — an opt-in Prettifier<T> is an ODR hazard, and the symptom is a segfault

Status: fixed 2026-08-16 (for Displayable); policy question open
Type: bug
Milestone: ppsink-migration
Progress: grep -rln 'struct Prettifier<' --include=*_pp.hpp xo-*/ 2>/dev/null | grep -v '/\.build/' | wc -l

Found 2026-08-16 by `utest.expression` segfaulting after RC converted
`TypeBlueprint` to `ref::Displayable`. 54 000 stack frames, cycling:

```
pretty<TypeBlueprint>              pretty.hpp:63     <- operator<< branch
  -> operator<<(PpSinkInserter&,)  pretty_ostream.hpp:70
    -> os << x
      -> operator<<(ostream&,)     TypeBlueprint_ostream.hpp:14
        -> tostr(x)                indentlog2/print/tostr.hpp:59
          -> PpSink::pp<>          pretty.hpp:149    <- back to the top
```

## Why "just include the header" is the wrong reading

The first diagnosis — *`type_unifier.cpp` forgot `Displayable_pp.hpp`* — was
wrong, and checking it is what found the real defect. `type_unifier.cpp`
included it, at line 9. The frame addresses say where each body came from:

```
#54098  Prettifier<Borrow<TypeBlueprint>>::print   0x7ffff7fa6481   libxo_expression.so
#54096  pretty<TypeBlueprint>  pretty.hpp:63       0x00411576       the utest EXECUTABLE
```

Two TUs instantiated `pretty<TypeBlueprint>`. The library's copy saw
`Prettifier<TypeBlueprint>` and took the `has_prettifier` branch; the utest's
copy did not, and took the `operator<<` branch. Same vague-linkage symbol, two
different bodies — **an ODR violation**. The dynamic linker kept one, the
utest's won, and the library's correct call sites started routing through it.

`xo::pp::pretty()` branches on `has_prettifier<T>` *at the point of
instantiation* (`xo-ppsink/include/xo/ppsink/pretty.hpp:56-71`). So any
`Prettifier<T>` that a TU can choose not to see changes the meaning of an inline
function that TUs are required to agree on. Nothing diagnoses it: no compile
error, no link error, no warning.

The recursion is what made it visible. Without it the failure mode is silent
wrong output — the flat render where the structured one was intended — which is
the same silent downgrade `issues/02` was about, arriving by a different route.

## Fixed for Displayable

`Prettifier<T> requires std::derived_from<T, ref::Displayable>` moved out of the
one-hour-old `Displayable_pp.hpp` and into
`xo-refcnt/include/xo/refcnt/Displayable.hpp` itself; the sidecar is deleted.
Now the Prettifier arrives with the type and cannot be half-visible.

That is the same choice `xo-alloc/issues/01` made for `Prettifier<gp<Object>>`,
which lives in `Object.hpp` beside the type. Two subsystems reached it
independently; it is the pattern.

Verified: `cmake --build .build -j` clean, `utest.expression` passes,
`xo-build --sweep` at `62 attempted: 34 ok, 28 with no tests, 0 failed,
0 skipped`.

## The policy question, which is open

**When is a `_pp.hpp` sidecar safe?** Only when the type has no *competing*
rendering path, so that a TU missing the sidecar gets a compile error rather
than a different body. Concretely:

| header | safe? | why |
|---|---|---|
| `Refcounted_pp.hpp` | yes, today | since `944c089c` + `explicit operator bool`, a TU without it fails to compile — there is no other path for `rp<T>` |
| `Displayable_pp.hpp` | **no** | `TypeBlueprint_ostream.hpp` was another path, so the missing case compiled |
| any future `X_pp.hpp` | ask | does `X` have an ostream inserter, or an implicit conversion, that would let the no-Prettifier branch compile? |

Note the asymmetry: `Refcounted_pp.hpp`'s safety is a *consequence* of
`issues/02`, not a property of the naming convention. Deleting `explicit` from
`operator bool` would silently make it unsafe again.

The stricter rule, and probably the right one: **`Prettifier<T>` belongs in the
header that declares `T`.** A sidecar is then only for types you do not own
(std::, third-party), where no TU can see a competing xo-authored inserter
either. Worth deciding before `ostream-containment` mints more `_pp.hpp` files.

## A second rule this exposes, about the `_ostream.hpp` bridges

`TypeBlueprint_ostream.hpp` implements `operator<<(ostream&, const T&)` as
`os << tostr(x)`, i.e. it renders *through ppsink*. ppsink's fallback renders
through `operator<<`. Those two form a cycle that is broken only by
`Prettifier<T>` being visible — so the bridge header is load-bearing for
termination, which is not obvious from reading it.

`xo-alloc/include/xo/alloc/alloc_ostream.hpp` has the same shape
(`FlatSink sink(os.rdbuf()); sink.pp(x);`) and is safe *only* because
`Prettifier<gp<Object>>` ships with `Object.hpp`. `type_unifier_ostream.hpp`
calls `x.print(os)` directly and cannot recurse at all.

So a bridge that renders via ppsink must include the type's Prettifier, or
render directly. With the Displayable fix above, every current bridge is
covered, but the constraint is undocumented in all of them.

## Worth considering: make it fail loudly instead of remembered

A `thread_local` reentrancy depth in `pretty_ostream.hpp`'s
`operator<<(PpSinkInserter&, const T&)`, asserting past depth ~2 in debug
builds, would have turned 54 000 frames into an immediate assert naming the
type. Cheap, and it catches the whole family rather than this instance. Not
implemented — costs a branch on the fallback path, so it wants RC's call.

## Done when

- a rule is written down for where `Prettifier<T>` may live, and the existing
  `_pp.hpp` headers are checked against it
- decided whether the reentrancy tripwire is worth its cost

## Related: how many Prettifiers may claim one type (2026-08-23)

This ticket asks WHERE a `Prettifier<T>` may be declared. The other half —
how many may claim the same type — turned up when xo-reactor gained constrained
partial specializations for its two roots, bringing the tree to three
(`Displayable`, `AbstractEventProcessor`, `Reactor`). They are disjoint only
because no class derives from two of those hierarchies; overlap is an ambiguous
partial specialization. See `.xo-backlog/xo-reactor/issues/01-aep-inherit-displayable.md`,
which also records why unifying them under Displayable is blocked.
