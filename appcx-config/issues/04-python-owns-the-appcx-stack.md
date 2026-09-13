# 04 — python should own its appcx stack, not a module-level static

Status: open
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

- **`AllocFlywheel::make_app` needs `py::keep_alive<0,1>`.** It takes
  `const FacetAppcx &` and flywheels retain it. Safe today only because the
  static guarantees the appcx outlives everything; once python owns the appcx,
  dropping `cx` while a flywheel lives dangles.
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

**Done when:**
- the `appcx_` grep above returns empty
- `configure()` returns a holder; the module-level `appcx()` accessor is gone,
  along with the test that only exercised its error path
- a python test builds TWO independent stacks in one process and shows their
  contexts are distinct objects -- the property the static made impossible, and
  therefore the proof this was worth doing
- `FacetRegistry::instance(capacity)` rejects a second, different capacity
