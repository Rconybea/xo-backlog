# 01 — python extension modules should be `xo.foo`, not `xo_pyfoo`

Status: open (design settled 2026-09-13; mechanism probed 2026-09-13, works)
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
the stack look like 18 unrelated libraries rather than one package.

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

**That count is 17, and the answer is 18.** `xo-pyutil` is INTERFACE-only and
defines no module of its own — but it *builds* one, from `example/ex1/`, through
the same macro, and installs it:

```bash
ls ~/local/lib/ | grep pyutilexample
#   xo_pyutilexample.cpython-312-x86_64-linux-gnu.so
```

Every count in this ticket is 18. The `xo-py*/src/*` glob that produced 17 is
recorded here because it is the obvious one to reach for and it is wrong — a
module built from an `example/` directory is still a module, and it goes through
`xo_pybind11_library()` like the rest.

### The C++ import sites need no edits

Every live cross-module import already routes through the generated macro, so
changing the template changes all of them at once:

```bash
grep -rn 'module_::import' xo-py*/src/*/*.cpp | grep -v '//'
#   xo-pyfacet/src/pyfacet/pyfacet.cpp:104:  import(PYINDENTLOG2_MODULE_NAME_STR);
```

One genuine literal, and it reads the macro. The bare-name calls that a naive
grep finds — `py::module_::import("pyreflect")` in six subsystems — are trailing
**comments** recording the pre-macro spelling, e.g.
`xo-pyreactor/src/pyreactor/pyreactor.cpp:29`. Left alone they would each be a
plausible-looking edit to a dead string.

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
ticket lands in `xo-cmake` (two macros) and in 18 `.hpp.in` templates; none of
it lands in `xo-pyutil`.

## Decisions (RC, 2026-09-13)

| question | decision | why |
|---|---|---|
| install destination | `${PREFIX}/lib/python/xo` | version-agnostic. Safe to share across python versions because the ABI tag is already in each filename and CPython accepts only the running interpreter's exact tag. `xo-python` sets `PYTHONPATH` explicitly, so nothing needs site-packages auto-discovery, and a version-stamped path would add a cmake-time query for no reader. |
| `xo_pyutilexample` | follows the rule → `xo.utilexample` | it is a demo of `xo_pybind11_library`; a demo that does not demonstrate the real layout teaches the wrong thing. Keeps the macro with one behaviour and no escape hatch. |
| staging | one commit, all 18 | the macro is a single switch and every C++ import site reads from it, so a half-converted tree protects no consumer and costs more than the whole change. |
| `xo_pyfoo` compatibility | break cleanly, no shims | nothing outside this tree imports them. Leaves nothing to deprecate later. |

## What changes

All of it inside `xo_pybind11_library()` — no subsystem `CMakeLists.txt` edits.

- cmake **target** name stays `xo_pyfacet`. Only the artifact moves:
  `OUTPUT_NAME facet`, `LIBRARY_OUTPUT_DIRECTORY <bindir>/python/xo`.
  **Verified 2026-09-13** — see the feasibility probe below. `OUTPUT_NAME` is
  untouched by pybind11: both of its tool paths set only `PREFIX`,
  `DEBUG_POSTFIX` and `SUFFIX` (`pybind11Tools.cmake:149` and
  `pybind11NewTools.cmake:334`, pybind11 2.13.6), so the two do not collide.
- install destination moves from `lib/` to `lib/python/xo/`. Today the `.so`
  sits beside `libxo_facet.so`:

  ```bash
  ls ~/local/lib/ | grep pyfacet
  #   xo_pyfacet.cpython-312-x86_64-linux-gnu.so
  ```

  `import xo_pyfacet` works only because the wrapper puts a **C++ library
  directory** on `PYTHONPATH` (`~/local/bin/xo-python:16`). That is worth fixing
  regardless of this ticket.
- `xo_emit_python_wrapper()` gets simpler: one `PYTHONPATH` entry (the parent of
  `xo/`) instead of one per module. In the umbrella that is 18 → 1. Its existing
  comment about a partial `PYTHONPATH` failing outright still applies and should
  survive the edit.
- 11 python import lines across the utests. These are the only import sites
  needing a hand edit; see above.

## Feasibility probe (2026-09-13, pybind11 2.13.6, python 3.12)

Built two throwaway modules to test the three mechanisms this ticket depends on
at once. All three work; nothing had to be worked around.

```cmake
pybind11_add_module(xo_pyfacet MODULE mod.cpp)     # target keeps the build name
set_target_properties(xo_pyfacet PROPERTIES
    OUTPUT_NAME facet
    LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/python/xo)
```

with `PYBIND11_MODULE(facet, m)` in `mod.cpp`, and a second module
`PYBIND11_MODULE(object2, m)` whose body does
`py::module_::import("xo.facet")` **at init**, emitted into a *different*
directory (`python2/xo`) to reproduce the split-portion case.

```
build/python/xo/facet.cpython-312-x86_64-linux-gnu.so     <- OUTPUT_NAME honoured
```

```bash
PYTHONPATH=build/python python3 -c 'import xo.facet; print(xo.facet.__name__)'
#   xo.facet

# the satellite-build shape: two portions, sibling import at module init
PYTHONPATH=build/python2:build/python python3 -c \
  'import xo.object2; print(xo.object2.sibling_says, len(list(__import__("xo").__path__)))'
#   xo.facet speaking 2
```

Confirmed by this:

1. **`OUTPUT_NAME` and `LIBRARY_OUTPUT_DIRECTORY` both apply** to a
   `pybind11_add_module` target, and the cmake target name stays free to remain
   `xo_pyfacet`.
2. **`PYBIND11_MODULE(facet, m)` is enough** — the `xo.` prefix comes from the
   directory, and `__name__` reports the full dotted `xo.facet`.
3. **A sibling import at module-init time crosses portions.** `xo.object2` in
   one portion imported `xo.facet` in another, during its own initialisation.
   That is the exact shape of `pyfacet.cpp:104` under a satellite build, and it
   was the mechanism most likely to fail.

Still inferred, not measured: that pybind11's cross-module type registry and
identity map are unaffected (see above) — the probe modules share no C++ types.
The existing `test_context_keeps_its_dependency_alive` spans two modules and
would notice, so this resolves itself at pilot time rather than needing its own
probe.

## Done when

- [ ] `xo_pybind11_library()` emits `xo/<name>.cpython-*.so` in build and install trees
- [ ] all 18 `.hpp.in` templates carry the `@SELF_MODULE@` / `@SELF_QUALNAME@` split
- [ ] no `__init__.py` is installed anywhere under `xo/`, and nothing in the tree
      creates one — this is the settled decision, not a default to revisit
- [ ] `xo-build --sweep` green, and `nix-build ci.nix -A xo-pyobject2` green —
      nix is the only check that exercises an installed layout as a consumer would
- [ ] a satellite build of one py subsystem imports its own fresh module *and*
      its installed dependencies in the same interpreter (the split-portion case
      measured above)

Progress: `ls xo-py*/src/*/*.hpp.in xo-pyutil/example/*/*.hpp.in | xargs grep -L SELF_QUALNAME | wc -l`

## Notes

Names after conversion carry their version suffixes unchanged — `xo.object2`,
`xo.reactor2`, `xo.indentlog2`. That is right: they are different modules from
`xo.object` and `xo.reactor`, which also still exist.

Not in scope, but adjacent: nothing in the tree ships a `.pyi` stub or any pure
python. Once `xo/` is a real directory that a subsystem installs into, stubs
have an obvious place to go.
