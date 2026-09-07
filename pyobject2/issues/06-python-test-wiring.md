# 06 — python tests for the xo-py* tier

Status: fixed 2026-09-07
Type: feature
Milestone: pyobject2

No `xo-py*` module has tests. The only pytest wiring in the tree is xo-cmake's:

```bash
find . -name 'test_*.py' -not -path './*/.build/*' | grep -v '^./xo-cmake'   # empty
sed -n '1,20p' xo-cmake/utest/CMakeLists.txt                                 # the add_test pattern
```

Most of this milestone's logic does not need python to be tested — handle
narrowing, recovery and root release are all c++ and belong in
`xo-facet/utest` (tickets 01, 02). What only python can assert is the binding
itself, and the lifetime coupling:

- a dropped python reference drops the strong root (`del o` lowers
  `strong_root_count()`)
- an object survives and still renders after an intervening allocation
- a repr missing a facet has no method for it

**Shape:** `examples/*.py` per module, following `xo-pyfacet/examples`, plus one
pytest module wired through ctest so `xo-build --sweep` runs it.

Note `--with-examples` already exists precisely because examples catch breakage
utests miss (`.xo-backlog/CONVENTIONS.md`); an example that imports the module
and renders one object is cheap coverage of the whole stack.

**Done when:** met 2026-09-07.  `xo-build --sweep` reports

```
xo-build: xo-pyobject2 ok (utest)
```

where xo-pyarena / xo-pyindentlog2 / xo-pyfacet still read `ok (utest:no-tests)`.
Stage 2 totals moved 36 -> 37 ok.  The suite ran in a STANDALONE subsystem
build, which is the case that matters: it proves the generated wrapper reaches
the installed dependencies, not just the umbrella's build tree.

## What landed

- `xo-pyobject2/utest/test_pyobject2.py` -- 13 cases: imports, config
  round-trip, `value()`, `repr()`, single- and shared-sink rendering, and the
  handle keeping its flywheel alive
- `xo-pyobject2/utest/CMakeLists.txt` -- runs them through
  `${CMAKE_BINARY_DIR}/xo-python`, which is the top of the build tree in BOTH
  contexts, so no umbrella/standalone branch is needed
- `xo_emit_python_wrapper()` now appends `$PREFIX/lib` for a standalone build,
  and each `xo-py*` emits its own wrapper when not part of the umbrella

## Deviations from the shape proposed above

**unittest, not pytest.**  pytest is not installed
(`python3 -c "import pytest"` fails), and xo-cmake/utest already uses
`python3 -m unittest discover`.  Following the precedent cost nothing and adds
no dependency.

**Configuration cases run in subprocesses.**  `configure_all()` is
process-global and one-shot, so "unconfigured raises", "second configure
raises", "supplied config is honoured" and "zero-reservation config rejected"
each need a pristine interpreter.  They spawn one via `sys.executable`, which
under the wrapper inherits the exported PYTHONPATH.

## NOT done here

- **`examples/*.py` per module.**  A worked example for Float exists but lives
  in a scratch directory, not the repo.
- **"a dropped python reference drops the strong root".**  Cannot be written
  yet: nothing releases the root and there is no `strong_root_count()` to
  observe.  That assertion belongs with `pyobject2/issues/02`, which adds both.
- **"a repr missing a facet has no method for it".**  Every bound repr
  currently has one; worth revisiting when a class without `APrintable` is
  bound.
