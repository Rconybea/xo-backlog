# 02 — `normalize`: scc JSON → row table

Status: open
Type: task

This is the **pluggable-backend seam**. Everything downstream sees only `Row`
tuples, so a `cloc` adapter could later emit the same table with
`complexity=0, generated=False` and nothing else would change.

**Files:**
- Modify: `xo-cmake/bin/xo-loc.in`
- Modify: `xo-cmake/utest/test_xo_loc.py`
- Create: `xo-cmake/utest/data/scc-sample.json`

**Interfaces:**
- Consumes: `load_xo_loc()` (ticket 01)
- Produces:
  - `Row = namedtuple("Row", "subsystem path language kind generated loc complexity")`
  - `KIND_BY_LANGUAGE: dict[str, str]`
  - `kind_of(language: str, generated: bool) -> str`
  - `normalize(raw: list) -> list[Row]`

---

- [ ] **Step 1: Create the fixture**

Create `xo-cmake/utest/data/scc-sample.json`. This is a trimmed capture of real
`scc --by-file --gen -f json` output — it doubles as the **pinned backend
contract**, so an scc upgrade that changes the JSON shape fails loudly here.

```json
[
  {
    "Name": "C++",
    "Files": [
      {"Location": "xo-gc/src/gc/Collector.cpp", "Code": 400, "Complexity": 44, "Generated": false},
      {"Location": "xo-imgui/include/imgui/imgui.cpp", "Code": 11375, "Complexity": 2100, "Generated": false}
    ]
  },
  {
    "Name": "C++ Header",
    "Files": [
      {"Location": "xo-type/include/xo/type/Type.hpp", "Code": 120, "Complexity": 3, "Generated": true},
      {"Location": "xo-gc/include/xo/gc/Collector.hpp", "Code": 90, "Complexity": 5, "Generated": false}
    ]
  },
  {
    "Name": "CMake",
    "Files": [
      {"Location": "xo-cmake/cmake/xo_macros/xo_cxx.cmake", "Code": 1066, "Complexity": 30, "Generated": false},
      {"Location": "xo-gc/CMakeLists.txt", "Code": 25, "Complexity": 1, "Generated": false}
    ]
  },
  {
    "Name": "Markdown",
    "Files": [
      {"Location": "xo-hashable2/README.md", "Code": 1, "Complexity": 0, "Generated": false}
    ]
  },
  {
    "Name": "Brainfuck",
    "Files": [
      {"Location": "xo-gc/weird.bf", "Code": 7, "Complexity": 0, "Generated": false}
    ]
  }
]
```

`Brainfuck` is deliberately absent from `KIND_BY_LANGUAGE` — it is the `Other`
fallback probe.

- [ ] **Step 2: Write the failing tests**

Append to `xo-cmake/utest/test_xo_loc.py`:

```python
import json


def load_fixture():
    with open(_HERE / "data" / "scc-sample.json") as f:
        return json.load(f)


class TestKindOf(unittest.TestCase):
    def setUp(self):
        self.xo_loc = load_xo_loc()

    def test_maps_languages_to_kinds(self):
        self.assertEqual(self.xo_loc.kind_of("C++", False), "Code")
        self.assertEqual(self.xo_loc.kind_of("C++ Header", False), "Code")
        self.assertEqual(self.xo_loc.kind_of("CMake", False), "Build")
        self.assertEqual(self.xo_loc.kind_of("Nix", False), "Build")
        self.assertEqual(self.xo_loc.kind_of("Markdown", False), "Docs")
        self.assertEqual(self.xo_loc.kind_of("JSON5", False), "Data")

    def test_unknown_language_falls_back_to_Other(self):
        self.assertEqual(self.xo_loc.kind_of("Brainfuck", False), "Other")

    def test_generated_overrides_the_language_mapping(self):
        # a generated .hpp is still C++, but its kind is Generated -- this is
        # why generated cannot be derived from a language lookup alone
        self.assertEqual(self.xo_loc.kind_of("C++ Header", True), "Generated")
        self.assertEqual(self.xo_loc.kind_of("Brainfuck", True), "Generated")


class TestNormalize(unittest.TestCase):
    def setUp(self):
        self.xo_loc = load_xo_loc()
        self.rows = self.xo_loc.normalize(load_fixture())

    def test_one_row_per_file(self):
        self.assertEqual(len(self.rows), 8)

    def test_subsystem_is_first_path_component(self):
        by_path = {r.path: r for r in self.rows}
        self.assertEqual(by_path["xo-gc/src/gc/Collector.cpp"].subsystem, "xo-gc")
        self.assertEqual(
            by_path["xo-imgui/include/imgui/imgui.cpp"].subsystem, "xo-imgui")

    def test_carries_loc_and_complexity(self):
        row = next(r for r in self.rows if r.path.endswith("Collector.cpp"))
        self.assertEqual(row.loc, 400)
        self.assertEqual(row.complexity, 44)

    def test_generated_flag_survives_into_the_row(self):
        row = next(r for r in self.rows if r.path.endswith("type/Type.hpp"))
        self.assertTrue(row.generated)
        self.assertEqual(row.kind, "Generated")
        self.assertEqual(row.language, "C++ Header")
```

- [ ] **Step 3: Run tests to verify they fail**

```bash
python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v
```

Expected: FAIL — `AttributeError: module 'xo_loc' has no attribute 'kind_of'`.

- [ ] **Step 4: Write minimal implementation**

Add to `xo-cmake/bin/xo-loc.in`, after the imports:

```python
import collections

# ----------------------------------------------------------------
# normalize: backend output -> a table nothing downstream can tell the
# backend from.  A cloc adapter would emit the same Row list with
# complexity=0 and generated=False.

Row = collections.namedtuple(
    "Row", "subsystem path language kind generated loc complexity")

KIND_BY_LANGUAGE = {
    # Code
    "C++": "Code",
    "C++ Header": "Code",
    "C Header": "Code",
    "C": "Code",
    "Objective C++": "Code",
    "Python": "Code",
    "JavaScript": "Code",
    "GLSL": "Code",
    "LLVM IR": "Code",
    # Build
    "CMake": "Build",
    "Nix": "Build",
    "Makefile": "Build",
    "Autoconf": "Build",
    "BASH": "Build",
    "Shell": "Build",
    "YAML": "Build",
    "INI": "Build",
    "Patch": "Build",
    # Docs
    "ReStructuredText": "Docs",
    "Markdown": "Docs",
    "Plain Text": "Docs",
    "License": "Docs",
    "HTML": "Docs",
    # Data
    "JSON5": "Data",
    "JSON": "Data",
    "Jinja": "Data",
    "SVG": "Data",
}


def kind_of(language, generated):
    """Artifact kind for a file.

    'generated' is a per-file property orthogonal to language -- a generated
    .hpp is still C++ -- so it overrides the language mapping rather than
    being looked up in it.
    """
    if generated:
        return "Generated"
    return KIND_BY_LANGUAGE.get(language, "Other")


def normalize(raw):
    """scc --by-file -f json output -> [Row]."""
    rows = []
    for lang in raw:
        language = lang["Name"]
        for f in lang["Files"]:
            path = f["Location"]
            generated = bool(f.get("Generated", False))
            rows.append(Row(subsystem=path.split("/")[0],
                            path=path,
                            language=language,
                            kind=kind_of(language, generated),
                            generated=generated,
                            loc=f["Code"],
                            complexity=f["Complexity"]))
    return rows
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v
```

Expected: PASS, 8 tests.

- [ ] **Step 6: Commit**

```bash
git add xo-cmake/bin/xo-loc.in xo-cmake/utest/
git commit -m "xo-loc: normalize scc json into a backend-neutral row table [FEATURE]"
```

## Comments
