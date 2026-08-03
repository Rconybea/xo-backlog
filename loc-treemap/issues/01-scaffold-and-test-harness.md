# 01 — Scaffold `xo-loc.in` and the Python test harness

Status: open
Type: task

**Why this is first:** the repo has **no Python tests at all** and `xo-cmake` has
no `enable_testing()`. Every later ticket needs somewhere to put a test, so this
ticket establishes it and proves it runs under `ctest`.

**Files:**
- Create: `xo-cmake/bin/xo-loc.in`
- Create: `xo-cmake/utest/test_xo_loc.py`
- Create: `xo-cmake/utest/CMakeLists.txt`
- Modify: `xo-cmake/CMakeLists.txt` (append test wiring at end of file)

**Interfaces:**
- Consumes: nothing
- Produces:
  - `XO_LOC_VERSION: str`
  - `xo-cmake/utest/test_xo_loc.py::load_xo_loc() -> module` — every later ticket's
    tests import the script through this helper

---

- [ ] **Step 1: Write the failing test**

Create `xo-cmake/utest/test_xo_loc.py`:

```python
"""Unit tests for xo-loc.

xo-loc lives at xo-cmake/bin/xo-loc.in.  The .in suffix is for CMake's
configure_file(), but the file is deliberately kept valid Python -- the only
substitution is inside a string literal -- so we can import it directly here
without running a configure step.
"""

import importlib.machinery
import importlib.util
import pathlib
import unittest

_HERE = pathlib.Path(__file__).resolve().parent
_XO_LOC = _HERE.parent / "bin" / "xo-loc.in"


def load_xo_loc():
    """Import xo-cmake/bin/xo-loc.in as a module named 'xo_loc'."""
    loader = importlib.machinery.SourceFileLoader("xo_loc", str(_XO_LOC))
    spec = importlib.util.spec_from_loader("xo_loc", loader)
    module = importlib.util.module_from_spec(spec)
    loader.exec_module(module)
    return module


class TestModuleLoads(unittest.TestCase):
    def test_script_is_valid_python_and_exposes_a_version(self):
        xo_loc = load_xo_loc()
        self.assertIsInstance(xo_loc.XO_LOC_VERSION, str)
        self.assertTrue(xo_loc.XO_LOC_VERSION)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run from the repo root:

```bash
python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v
```

Expected: FAIL — `FileNotFoundError` for `xo-cmake/bin/xo-loc.in`.

- [ ] **Step 3: Write minimal implementation**

Create `xo-cmake/bin/xo-loc.in`:

```python
#!/usr/bin/env python3
"""xo-loc - treemap of the xo subsystem world, area proportional to lines of code.

Sibling to xo-deps.  Counts the tree live with scc on every invocation; there is
deliberately no checked-in snapshot data file.

This file is configure_file()'d by xo-cmake, but is kept valid Python as-is:
the only @VAR@ substitution sits inside a string literal, so the unit tests can
import it straight from the source tree.
"""

import sys

XO_LOC_VERSION = "@PROJECT_VERSION@"


def main(argv=None):
    argv = sys.argv[1:] if argv is None else argv
    if "--version" in argv:
        print("xo-loc " + XO_LOC_VERSION)
        return 0
    print("xo-loc: not implemented yet", file=sys.stderr)
    return 1


if __name__ == "__main__":
    sys.exit(main())
```

Note: unconfigured, `XO_LOC_VERSION` is the literal `"@PROJECT_VERSION@"` — a
non-empty string, which is all the test asserts.

- [ ] **Step 4: Run test to verify it passes**

```bash
python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v
```

Expected: PASS, 1 test.

- [ ] **Step 5: Wire the tests into CMake**

Create `xo-cmake/utest/CMakeLists.txt`:

```cmake
# xo-cmake/utest/CMakeLists.txt
#
# Python unit tests for the scripts in xo-cmake/bin.
#

find_program(XO_PYTHON3_EXECUTABLE NAMES python3)

if (NOT XO_PYTHON3_EXECUTABLE)
    message(FATAL_ERROR "xo-cmake/utest: python3 not found, needed for xo-loc tests")
endif()

add_test(
    NAME utest.xo-loc
    COMMAND ${XO_PYTHON3_EXECUTABLE} -m unittest discover
            -s ${CMAKE_CURRENT_SOURCE_DIR} -p "test_*.py" -v
)

# end CMakeLists.txt
```

Append to the end of `xo-cmake/CMakeLists.txt`:

```cmake
if (ENABLE_TESTING)
    enable_testing()
    add_subdirectory(utest)
endif()
```

`find_program` rather than `find_package(Python3 ...)` because
`xo-cmake/CMakeLists.txt:1` declares `cmake_minimum_required(VERSION 3.10)` and
CMake's `FindPython3` module needs 3.12.

The `if (ENABLE_TESTING)` guard matches the repo convention — see
`xo-arena/utest/CMakeLists.txt:16`.

- [ ] **Step 6: Verify it runs under ctest**

```bash
cmake -S xo-cmake -B /tmp/xo-loc-build -DENABLE_TESTING=ON
ctest --test-dir /tmp/xo-loc-build --output-on-failure
```

Expected: `utest.xo-loc` passes, `100% tests passed, 0 tests failed out of 1`.

- [ ] **Step 7: Commit**

```bash
git add xo-cmake/bin/xo-loc.in xo-cmake/utest/ xo-cmake/CMakeLists.txt
git commit -m "xo-cmake: xo-loc skeleton + python utest harness [FEATURE]"
```

## Comments
