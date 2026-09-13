# 04 — python should own its appcx stack, not a module-level static

Status: open (python side done 2026-09-13; the named singletons remain)
Type: refactor

Each py module keeps its context in a process-global `unique_ptr`, reachable
through a module-level accessor:

```bash
grep -rn 'static std::unique_ptr<.*Appcx> appcx_' xo-py*/src/ | grep -v '/\.build/'
```

Replace with python ownership, so the stack is built incrementally and each
level's lifetime belongs to the caller that holds it.

## Why this is here

**Not** because the static is load-bearing and needs replacing. It is a
residue. RC's reading, which the evidence supports: the original design assumed
python setup happens automatically at import, and the module-level storage is
what that assumed. When the trigger moved to an explicit `configure()`, the
storage stayed because nothing re-asked whether it was needed.

The comment in `xo-pyindentlog2` reads as a rebuttal to the earlier design
rather than a justification of the current one:

```bash
grep -n 'NOT constructed at import' -A 4 xo-pyindentlog2/src/pyindentlog2/pyindentlog2.cpp
```

And the accessor serves nothing: `f.appcx()` appears exactly once across all
python, inside a subprocess source string asserting that it THROWS when
unconfigured. Every real caller does `cx = f.configure_all()` and threads `cx`.

```bash
grep -rn 'appcx()' xo-pyobject2/utest/*.py xo-pyobject2/example/*/*.py | grep -v indentlog2_appcx
```

The underlying tension, worth stating because it will recur: **an import
carries no arguments.** Automatic-at-import and caller-configurable are
mutually exclusive. The prior resolved it toward ergonomics by reflex; the code
then moved toward configurability without the storage following.

## Shape

pybind's default holder for `py::class_<T>` is `std::unique_ptr<T>`, so python
can own a heap-allocated Appcx with no change to the C++ type:

```cpp
m.def("configure",
      [](const FacetConfig & cfg, const Indentlog2Appcx & il) {
          return std::make_unique<FacetAppcx>(cfg, il);
      },
      py::arg("config"), py::arg("indentlog2_appcx"),
      py::keep_alive<0, 2>());      // returned appcx keeps `il` alive
```

`keep_alive<0,2>` makes the dependency edge a PYTHON reference, which is what
lets the stack be built a level at a time and collected as a unit.

### Why no intrusive refcount

`rp<>` would be the obvious holder -- `AllocFlywheel` in the same file uses it
-- and it is the wrong answer here. `AppContext<Tags...>` stores each Appcx as
a BASE SUBOBJECT:

```bash
grep -n 'class AppContext' -A 6 xo-subsys/include/xo/subsys/AppContext.hpp
grep -n 'chain_for' xo-subsys/include/xo/subsys/AppContext.hpp | head
```

so by-value membership is structural, and an intrusive count fights it.
`unique_ptr` holder plus `keep_alive` gets python ownership without giving the
type a refcount, so the same class serves both worlds.

Non-copyable and non-movable are both fine: the holder never copies, and
`make_unique` constructs in place.

### Why python must not use AppContext

`AppContext<A>` and `AppContext<A,B>` are different c++ types, so an
incrementally-grown one changes type at every step and invalidates whatever
python was holding. The per-subsystem constructor is the one to use, and it
already exists for exactly this:

```cpp
FacetAppcx(const FacetConfig & cfg, const Indentlog2Appcx & ilog2_appcx);  // standalone
template <typename Deps> FacetAppcx(Deps & deps, const FacetConfig & cfg); // chain
```

The standalone one takes a REFERENCE to the level below, which is what
incremental wiring needs. Two constructors, two worlds; the split is already
right.

## Consequences to handle

- **`AllocFlywheel::make_app` needs `py::keep_alive<0,1>`.** DONE. It stores
  `const FacetAppcx & facet_appcx_`, so a flywheel must not outlive the context
  it came from. This was latent until the conversion and became live with it --
  while the module owned the context it outlived everything by construction.
- **Never two owners for one Appcx.** One living inside a c++ `AppContext`
  reaches python by `reference`/`reference_internal` only, never
  `take_ownership`. The current code draws this line correctly; keep it.
- **The one-shot guard must move DOWN, not be deleted.** `configure()`'s throw
  is currently the only thing stopping a second call from silently ignoring the
  config it was handed. What is actually one-shot is
  `FacetRegistry::instance(capacity)`, which is a function-local static:

  ```bash
  sed -n '65,68p' xo-facet/include/xo/facet/FacetRegistry.hpp
  #   static FacetRegistry & instance(uint32_t hint_max_capacity = 1024) {
  #       static FacetRegistry s_instance(hint_max_capacity);
  ```

  so the argument is consumed on the FIRST call and silently ignored on every
  later one -- a second caller gets a registry sized by whoever got there
  first. `TypeRegistry` is the same. That is precisely what pyfacet's throw is
  compensating for, one level up. Deleting the guard along with the static is
  therefore a regression; the fix is for `instance(capacity)` to reject a
  second, differing capacity itself, at which point the python-level guard is
  redundant rather than merely misplaced.
- **The composition-level static_assert is lost.** `AppContext::cx<Tag>()`
  static_asserts that a tag belongs to the composition. The incremental form
  checks each EDGE instead, by argument type -- a `PrintJsonAppcx` cannot be
  passed where `Indentlog2Appcx&` is wanted. Weaker, and the actual price of
  building incrementally; checked rather than assumed, so acceptable.

## Object identity is already preserved, and stays so

pybind keeps a global address -> instance map, so any non-copying return of the
same address finds the existing python object -- across modules too. Measured
2026-09-13, before any of this:

```python
cx = f.configure_all()
cx is f.appcx()                                  # True
cx.indentlog2_appcx() is il.appcx()              # True, across modules
```

The one thing that would break it is returning an Appcx **by value**: the
default policy for that is copy/move, giving a fresh wrapper per call. So an
owning Appcx must never be returned by value -- which rules out "return by
value and let python own it" as the route to python ownership. The holder is
the route.

**Files:** `xo-pyindentlog2/src/pyindentlog2/pyindentlog2.cpp`,
`xo-pyfacet/src/pyfacet/pyfacet.cpp`, and any py module that grows an appcx
after them.

## What landed, 2026-09-13

Both `xo-pyindentlog2` and `xo-pyfacet` converted; the `appcx_` grep is empty.
Each module's `configure_once()` returns `std::unique_ptr<Appcx>`, both
`appcx()` accessors are gone, and what remains of the one-shot is a
function-local `bool` -- module OWNERSHIP eliminated, module state reduced to a
flag.

The guard's comment now names the singletons that make it necessary
(`PpSinkFactory`/`TempArena` for indentlog2, `FacetRegistry`/`TypeRegistry` for
facet), which is the concrete retirement condition: the flag goes when THOSE
go.

**`configure_all` had a dangling bug the moment pyindentlog2 converted.** It
cast a TEMPORARY `py::object` to `Indentlog2Appcx &`; once that object owns the
context, it dies at the end of the full expression. Fixed by holding the
`py::object` in a local and calling `py::detail::keep_alive_impl(f_obj, il_obj)`
by hand -- the patient is a local here, not an argument, so the declarative form
does not apply.

### Testing keep_alive: two wrong ways first

Worth recording, because the obvious tests are worthless:

1. **Reading the context after dropping the handle passes either way.** Without
   the keep_alive the c++ object is freed, and reading it is a use-after-free
   that returns the right bytes. Measured, not assumed.
2. **python's gc cannot see the reference.** pybind stores the patient in its
   own internals map, so `gc.get_objects()` and `gc.get_referents(cx)` show
   nothing.

What discriminates is a **weakref on the python object**: alive with the
mechanism, collected without. Each keep_alive now has a test that goes red when
that one mechanism is removed, and only that one -- checked for all three
(`configure`, `configure_all`, `make_app`).

A third failure mode is worth naming: while xo_pyindentlog2 still owned its
context, a correct test of pyfacet's keep_alive STILL passed, because dropping
a non-owning wrapper collects nothing. Converting one module without the other
leaves a mechanism that cannot be verified -- which is why they went together.

**Done when:**
- ~~the `appcx_` grep above returns empty~~ -- done
- ~~`configure()` returns a holder; the module-level `appcx()` accessor is
  gone~~ -- done; the test that only exercised its error path is replaced by one
  asserting neither module has the accessor
- ~~each keep_alive has a test that fails without it~~ -- done, via weakref
- `FacetRegistry::instance(capacity)` rejects a second, different capacity.
  STILL OPEN, and the last item: a c++ caller gets no warning today, and the
  python guard is what hides it.

**Not achievable at this layer, and removed from scope:** a python test building
two INDEPENDENT stacks. `FacetAppcx` is a view over `FacetRegistry::instance()`
/ `TypeRegistry::instance()`, so a second context would be a second view of one
registry with its config silently ignored -- which is exactly what
`configure_once` refuses. Independent stacks are gated on those singletons
going, not on anything in this ticket. Recorded rather than quietly dropped:
the original done-when assumed python ownership implied independence, and it
does not.
