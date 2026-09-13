# 01 — python extension modules should be `xo.foo`, not `xo_pyfoo`

Status: open (design settled 2026-09-13; implementation unverified)
Type: design / build
Raised: RC, 2026-09-13

Today every pybind11 module is a top-level name mirroring its cmake target:

```python
import xo_pyfacet, xo_pyobject2
```

It should be

```python
import xo.facet, xo.object2
```

`xo_pyfoo` is a *build* identifier — it exists so the cmake target is globally
unique and so `xo_pybind11_dependency()` has something to key on. Leaking it
into the import namespace makes python users say the internal name, and makes
the stack look like 17 unrelated libraries rather than one package.

## What python actually requires

Three things, and only three:

1. a directory `xo/` on `sys.path` containing `facet.cpython-312-x86_64-linux-gnu.so`
2. `PYBIND11_MODULE(facet, m)` — CPython looks for `PyInit_facet`, keyed on the
   **last dotted component only**. The `xo.` prefix comes entirely from where
   the file sits, never from the init symbol.
3. sibling imports spelled `py::module_::import("xo.indentlog2")`
   (`xo-pyfacet/src/pyfacet/pyfacet.cpp:104` is the one live instance today)

Inferred, not verified: pybind11's cross-module type registry and its
address→instance identity map key on type/address, not on module name, so the
identity machinery landed under `appcx-config/04` is untouched by this. Worth
confirming with the existing `test_context_keeps_its_dependency_alive` once a
pilot module is renamed — that test spans two modules, so it would notice.

## What makes it cheap

The module name is **already indirect**. `xo_pybind11_library()` generates a
per-subsystem header from a `.hpp.in` template
(`xo-cmake/cmake/xo_macros/xo_cxx.cmake:1835`):

```
#define PYFACET_MODULE_NAME() @SELF_LIB@
#define PYFACET_MODULE_NAME_STR "@SELF_LIB@"
```

One value currently serves both roles. They separate:

- `@SELF_MODULE@` = `facet` — for `PYBIND11_MODULE`
- `@SELF_QUALNAME@` = `xo.facet` — for `import` and `_STR`

Count of modules, and of templates to edit:

```bash
grep -rl PYBIND11_MODULE xo-py*/src/*/*.cpp | wc -l   # 17, 2026-09-13
ls xo-py*/src/*/*.hpp.in | wc -l                       # 17
```

`xo-pyutil` is INTERFACE-only and defines no module — it is not one of the 17.

## Decided 2026-09-13 (RC): `xo` is a PEP 420 namespace package

**There is no `xo/__init__.py`, and none is to be added.**

PEP 420 (python 3.3) made a directory a package without an `__init__.py`. The
two kinds behave differently when the same package name appears at more than one
`sys.path` entry:

- **regular package** (has `__init__.py`) — import finds the *first* one and
  stops. Any same-named directory further along `sys.path` is invisible.
- **namespace package** (no `__init__.py`) — import scans *every* `sys.path`
  entry and merges all matching directories into one package. Each contributing
  directory is a *portion*.

This is not a tidiness preference. A regular package **breaks the
satellite-build workflow**; a namespace package does not.

`xo_emit_python_wrapper()` prepends the build tree's module directories to
`PYTHONPATH` and appends the install prefix, deliberately, so a freshly built
module beats an installed one (`xo-cmake/cmake/xo_macros/xo_cxx.cmake:2137-2149`
and its comment). After this change that means two `sys.path` entries both
offering `xo/`: the build tree with the one module being worked on, the install
tree with the other sixteen.

Measured 2026-09-13, python 3.12:

```bash
cd $(mktemp -d) && mkdir -p build/xo inst/xo
echo 'name="from build"' > build/xo/facet.py
echo 'name="from inst"'  > inst/xo/reflect.py

# PEP 420: no __init__.py anywhere
PYTHONPATH=build:inst python3 -c 'import xo.facet, xo.reflect; print(xo.facet.name, xo.reflect.name)'
#   from build from inst

touch build/xo/__init__.py inst/xo/__init__.py
PYTHONPATH=build:inst python3 -c 'import xo.facet, xo.reflect; print(xo.facet.name, xo.reflect.name)'
#   ModuleNotFoundError: No module named 'xo.reflect'
```

A regular package resolves to the **first** `__init__.py` and the other portion
becomes invisible. A namespace package merges both portions.

PEP 420 also dissolves an ownership question rather than answering it: with an
`__init__.py`, some subsystem has to install it, and that subsystem must sit
below all 17 in the levelization — a constraint the flat `lib/` layout does not
currently impose on anyone. With a namespace package each subsystem installs its
own `.so` into `xo/` independently and satellite installs compose.

### Why the reversal option is closed, not merely unused

An earlier draft of this ticket called the choice reversible — "add
`xo/__init__.py` later if `xo.configure()` ever wants a home". Kept here because
it is the obvious thing to reach for, and it is wrong on both halves.

**There is nothing for a package-level `configure()` to do.** `configure_all()`
already exists, at the top of the stack rather than at package level
(`xo-pyfacet/src/pyfacet/pyfacet.cpp:98`); it configures xo-facet *and*
xo-indentlog2 in one call, and its own docstring says why that is legitimate:

> This reaches into `xo_pyindentlog2`, but not silently: taking an
> `Indentlog2Config` is what says so. The argument is the evidence — which is
> the difference between a convenience and a secret.

An `xo.configure()` would reach across every module in the stack with no
dependency edge and no argument justifying it: the secret, not the convenience.
That is the same design error as the module-level `PyFacetAppcx` static removed
under `appcx-config/04`, relocated rather than avoided.

**And `__init__.py`'s one real capability is one this tree rejects** — running
code at `import xo` time, before any submodule loads. From the same file:

> Defaults are materialized HERE, per call, rather than as pybind default
> arguments: those are evaluated once when the module is imported, and an import
> should do nothing but register types.

So a regular package would cost two dependency edges (`xo-pyprocess`,
`xo-pyreactor2` onto whichever subsystem installed the file — see below) that
mean only "installs into the same directory", in exchange for a capability the
design does not want.

### Which also settles the owner question

With no `__init__.py` there is nothing to own, so no subsystem needs to sit
below all 17. `xo-pyutil` was the candidate — 15 of the 17 modules include
`pyutil.hpp`, and it is the lowest py subsystem at position 11:

```bash
for d in xo-py*/; do s=${d%/}; grep -q PYBIND11_MODULE $s/src/*/*.cpp 2>/dev/null || continue
  grep -q pyutil $s/src/*/*.cpp && echo "$s yes" || echo "$s NO"; done
#   xo-pyprocess NO, xo-pyreactor2 NO, the other 15 yes   (2026-09-13)
```

It stays a header-only C++ subsystem with no packaging role. The work in this
ticket lands in `xo-cmake` (two macros) and in 17 `.hpp.in` templates; none of
it lands in `xo-pyutil`.

## What changes

All of it inside `xo_pybind11_library()` — no subsystem `CMakeLists.txt` edits.

- cmake **target** name stays `xo_pyfacet`. Only the artifact moves:
  `OUTPUT_NAME facet`, `LIBRARY_OUTPUT_DIRECTORY <bindir>/python/xo`.
  Unverified: that `pybind11_add_module` respects `OUTPUT_NAME` cleanly
  alongside the `SUFFIX` it sets. Check this first on one module — it is the
  cheapest thing that could sink the approach.
- install destination moves from `lib/` to a real site dir. Today the `.so`
  sits beside `libxo_facet.so`:

  ```bash
  ls ~/local/lib/ | grep pyfacet
  #   xo_pyfacet.cpython-312-x86_64-linux-gnu.so
  ```

  `import xo_pyfacet` works only because the wrapper puts a **C++ library
  directory** on `PYTHONPATH` (`~/local/bin/xo-python:16`). That is worth fixing
  regardless of this ticket.
- `xo_emit_python_wrapper()` gets simpler: one `PYTHONPATH` entry (the parent of
  `xo/`) instead of one per module. In the umbrella that is 17 → 1. Its existing
  comment about a partial `PYTHONPATH` failing outright still applies and should
  survive the edit.
- 11 python import lines across the utests.

## Done when

- [ ] `xo_pybind11_library()` emits `xo/<name>.cpython-*.so` in build and install trees
- [ ] all 17 `.hpp.in` templates carry the `@SELF_MODULE@` / `@SELF_QUALNAME@` split
- [ ] no `__init__.py` is installed anywhere under `xo/`, and nothing in the tree
      creates one — this is the settled decision, not a default to revisit
- [ ] `xo-build --sweep` green, and `nix-build ci.nix -A xo-pyobject2` green —
      nix is the only check that exercises an installed layout as a consumer would
- [ ] a satellite build of one py subsystem imports its own fresh module *and*
      its installed dependencies in the same interpreter (the split-portion case
      measured above)

Progress: `ls xo-py*/src/*/*.hpp.in | xargs grep -L SELF_QUALNAME | wc -l`

## Notes

Names after conversion carry their version suffixes unchanged — `xo.object2`,
`xo.reactor2`, `xo.indentlog2`. That is right: they are different modules from
`xo.object` and `xo.reactor`, which also still exist.

Not in scope, but adjacent: nothing in the tree ships a `.pyi` stub or any pure
python. Once `xo/` is a real directory that a subsystem installs into, stubs
have an obvious place to go.
