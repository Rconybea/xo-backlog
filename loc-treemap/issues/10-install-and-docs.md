# 10 — CMake install wiring and documentation

Status: open
Type: task

Makes `xo-loc` a real installed tool alongside `xo-deps`.

**Files:**
- Modify: `xo-cmake/CMakeLists.txt` (configure_file + CHMOD + install)
- Modify: `xo-cmake/README.md`
- Create: `xo-cmake/utest/test_xo_loc_configured.py`

**Interfaces:**
- Consumes: the finished `xo-cmake/bin/xo-loc.in`
- Produces: `${CMAKE_INSTALL_BINDIR}/xo-loc`

---

- [ ] **Step 1: Write the failing test**

Create `xo-cmake/utest/test_xo_loc_configured.py`:

```python
"""Checks that survive configure_file() substitution."""

import pathlib
import re
import unittest

_HERE = pathlib.Path(__file__).resolve().parent
_XO_LOC_IN = _HERE.parent / "bin" / "xo-loc.in"


class TestConfigureSafety(unittest.TestCase):
    def setUp(self):
        self.text = _XO_LOC_IN.read_text()

    def test_every_at_placeholder_sits_inside_a_string_literal(self):
        # the file must stay importable straight from the source tree, which is
        # how the unit tests load it -- so no bare @VAR@ tokens
        for match in re.finditer(r"@[A-Z0-9_]+@", self.text):
            line = self.text[:match.start()].count("\n") + 1
            source = self.text.splitlines()[line - 1]
            self.assertRegex(
                source, r'"@[A-Z0-9_]+@"',
                "line %d: %r must keep its @VAR@ inside a string literal"
                % (line, source))

    def test_has_a_shebang(self):
        self.assertTrue(self.text.startswith("#!/usr/bin/env python3"))
```

- [ ] **Step 2: Run test to verify it passes already**

```bash
python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v
```

Expected: PASS — this test pins an invariant the earlier tickets already
established, so it should be green immediately. If it fails, an earlier ticket
introduced a bare `@VAR@` and broke importability; fix that first.

- [ ] **Step 3: Add the configure_file and CHMOD**

In `xo-cmake/CMakeLists.txt`, immediately after the existing `xo-deps` block
(`xo-cmake/CMakeLists.txt:55-61`), add:

```cmake
configure_file(
    ${PROJECT_SOURCE_DIR}/bin/xo-loc.in
    ${PROJECT_BINARY_DIR}/xo-loc
    @ONLY
    )
file(CHMOD ${PROJECT_BINARY_DIR}/xo-loc
    PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE
                GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
```

- [ ] **Step 4: Add it to the install list**

In the same file, add `"${PROJECT_BINARY_DIR}/xo-loc"` to the `install(FILES ...)`
list that already carries `xo-deps` — immediately after the `xo-deps` line:

```cmake
    "${PROJECT_BINARY_DIR}/xo-deps"
    "${PROJECT_BINARY_DIR}/xo-loc"
```

- [ ] **Step 5: Verify configure, install and run**

```bash
cmake -S xo-cmake -B /tmp/xo-loc-build -DENABLE_TESTING=ON \
      -DCMAKE_INSTALL_PREFIX=/tmp/xo-loc-prefix
cmake --build /tmp/xo-loc-build
ctest --test-dir /tmp/xo-loc-build --output-on-failure
cmake --install /tmp/xo-loc-build

/tmp/xo-loc-prefix/bin/xo-loc --version
cd /tmp && /tmp/xo-loc-prefix/bin/xo-loc \
    --root=$HOME/proj/xo-umbrella2 --format=csv --output=/tmp/installed.csv
head -3 /tmp/installed.csv
```

Expected: `--version` prints a real version rather than `@PROJECT_VERSION@`, and
the installed binary produces the same csv as the source-tree run.

- [ ] **Step 6: Verify root auto-discovery from a subdirectory**

```bash
cd ~/proj/xo-umbrella2/xo-gc/src/gc
/tmp/xo-loc-prefix/bin/xo-loc --format=json --output=/tmp/from-deep.json
python3 -c "import json;d=json.load(open('/tmp/from-deep.json'));print(d['total_loc'])"
```

Expected: the same `total_loc` as running from the repo root — the walk is
rooted at the umbrella regardless of `$PWD`. A different (much larger) number
means `.build` artifacts leaked in and root discovery is broken.

- [ ] **Step 7: Document it**

Add to `xo-cmake/README.md`, in the same style as the existing `xo-deps` entry:

```markdown
### xo-loc

Treemap of the xo subsystem world: one tile per subsystem, area proportional to
lines of code, color by artifact kind.  Sibling to `xo-deps`.

    xo-loc                              # svg of the whole tree
    xo-loc --format=html                # interactive: hover for numbers, click to focus
    xo-loc --color=complexity           # shade Code tiles by complexity per kLOC
    xo-loc --format=csv                 # the underlying table
    xo-loc xo-gc xo-arena               # just these subsystems

Requires `scc` (>= 3.5.0) on PATH; the tree is counted live on every run, so
there is no snapshot file to go stale.

By default the map shows hand-written code: vendored third-party code and
generated code are excluded, and every exclusion is reported on stderr.  Use
`--include-vendored` and `--include-generated` to put them back.
```

- [ ] **Step 8: Commit**

```bash
git add xo-cmake/CMakeLists.txt xo-cmake/README.md xo-cmake/utest/
git commit -m "xo-cmake: install xo-loc alongside xo-deps [FEATURE]"
```

## Comments
