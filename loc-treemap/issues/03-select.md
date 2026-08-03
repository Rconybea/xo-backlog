# 03 — `select`: exclusions, aggregation, and the honesty report

Status: open
Type: task

Applies every exclusion the spec decided, aggregates to `(subsystem, kind)`, and
— critically — **records what it hid**. A treemap that silently drops a third of
the tree is this design's main failure mode, so the report is a first-class
return value, not a print statement.

**Files:**
- Modify: `xo-cmake/bin/xo-loc.in`
- Modify: `xo-cmake/utest/test_xo_loc.py`

**Interfaces:**
- Consumes: `Row`, `normalize` (ticket 02)
- Produces:
  - `DEFAULT_VENDORED: tuple[str, ...]`
  - `Cell = namedtuple("Cell", "loc complexity nfiles languages")` — `languages`
    is `dict[str, int]` (LOC by language) and **defaults to `None`**, so tests
    may build cells with three positional args. It exists because the spec
    requires the tooltip to carry a language breakdown, which is otherwise lost
    at aggregation.
  - `Selection = namedtuple("Selection", "groups notes")` where
    `groups: dict[str, dict[str, Cell]]` keyed `subsystem -> kind -> Cell`
  - `select(rows, known_subsystems, subsystems=None, include_generated=False, include_vendored=False, vendored=DEFAULT_VENDORED, kinds=None, min_loc=20) -> Selection`
  - `subsystem_loc(groups) -> dict[str, int]`

---

- [ ] **Step 1: Write the failing tests**

Append to `xo-cmake/utest/test_xo_loc.py`:

```python
ALL_SUBSYSTEMS = {"xo-gc", "xo-imgui", "xo-type", "xo-cmake", "xo-hashable2"}


class TestSelect(unittest.TestCase):
    def setUp(self):
        self.xo_loc = load_xo_loc()
        self.rows = self.xo_loc.normalize(load_fixture())

    def sel(self, **kw):
        return self.xo_loc.select(self.rows, ALL_SUBSYSTEMS, **kw)

    def test_default_view_drops_vendored_and_generated(self):
        groups = self.sel().groups
        # imgui.cpp (11375 LOC) is vendored -> xo-imgui keeps nothing here
        self.assertNotIn("xo-imgui", groups)
        # xo-type's only file is generated -> nothing left
        self.assertNotIn("xo-type", groups)
        self.assertIn("xo-gc", groups)

    def test_include_vendored_restores_imgui(self):
        groups = self.sel(include_vendored=True).groups
        self.assertEqual(groups["xo-imgui"]["Code"].loc, 11375)

    def test_include_generated_restores_it_as_its_own_kind(self):
        groups = self.sel(include_generated=True).groups
        self.assertEqual(groups["xo-type"]["Generated"].loc, 120)

    def test_aggregates_by_subsystem_and_kind(self):
        gc = self.sel().groups["xo-gc"]
        self.assertEqual(gc["Code"].loc, 490)        # 400 .cpp + 90 .hpp
        self.assertEqual(gc["Code"].complexity, 49)  # 44 + 5
        self.assertEqual(gc["Code"].nfiles, 2)
        self.assertEqual(gc["Build"].loc, 25)        # CMakeLists.txt
        self.assertEqual(gc["Other"].loc, 7)         # weird.bf

    def test_keeps_the_language_breakdown_for_the_tooltip(self):
        gc = self.sel().groups["xo-gc"]
        self.assertEqual(gc["Code"].languages, {"C++": 400, "C++ Header": 90})

    def test_empty_placeholder_is_reported_not_drawn(self):
        selection = self.sel()
        self.assertNotIn("xo-hashable2", selection.groups)
        self.assertTrue(
            any("xo-hashable2" in n for n in selection.notes),
            "an excluded subsystem must be named in the report")

    def test_min_loc_tests_total_loc_not_code_loc(self):
        # REGRESSION: xo-cmake is 100% Build and has zero Code LOC, but it is a
        # real subsystem.  A 'zero code LOC' emptiness test wrongly drops it.
        groups = self.sel().groups
        self.assertIn("xo-cmake", groups)
        self.assertEqual(groups["xo-cmake"]["Build"].loc, 1066)

    def test_non_xo_paths_are_ignored(self):
        rows = self.rows + [self.xo_loc.Row(
            subsystem="pkgs", path="pkgs/xo-gc.nix", language="Nix",
            kind="Build", generated=False, loc=99, complexity=0)]
        groups = self.xo_loc.select(rows, ALL_SUBSYSTEMS).groups
        self.assertNotIn("pkgs", groups)

    def test_positional_subsystems_restrict_the_output(self):
        groups = self.sel(subsystems=["xo-gc"]).groups
        self.assertEqual(set(groups), {"xo-gc"})

    def test_kinds_filter(self):
        groups = self.sel(kinds=["Build"]).groups
        self.assertEqual(set(groups["xo-gc"]), {"Build"})

    def test_notes_quantify_every_exclusion(self):
        notes = " | ".join(self.sel().notes)
        self.assertIn("11375", notes)  # vendored LOC hidden
        self.assertIn("120", notes)    # generated LOC hidden


class TestSubsystemLoc(unittest.TestCase):
    def test_totals_across_kinds(self):
        xo_loc = load_xo_loc()
        rows = xo_loc.normalize(load_fixture())
        groups = xo_loc.select(rows, ALL_SUBSYSTEMS).groups
        totals = xo_loc.subsystem_loc(groups)
        self.assertEqual(totals["xo-gc"], 522)  # 490 Code + 25 Build + 7 Other
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v
```

Expected: FAIL — `AttributeError: module 'xo_loc' has no attribute 'select'`.

- [ ] **Step 3: Write minimal implementation**

Add to `xo-cmake/bin/xo-loc.in`:

```python
# ----------------------------------------------------------------
# select: apply exclusions, aggregate to (subsystem, kind).
#
# Every exclusion is counted and reported.  A treemap that silently drops a
# third of the tree is worse than no treemap.

DEFAULT_VENDORED = ("xo-imgui/include/imgui/",)

Cell = collections.namedtuple("Cell", "loc complexity nfiles languages")
# `languages` is optional so callers (and tests) can build a cell positionally
Cell.__new__.__defaults__ = (None,)

Selection = collections.namedtuple("Selection", "groups notes")


def select(rows, known_subsystems,
           subsystems=None, include_generated=False, include_vendored=False,
           vendored=DEFAULT_VENDORED, kinds=None, min_loc=20):
    notes = []

    # 1. xo-* directories only.  The root walk also sees pkgs/, docs/,
    #    README.md and xo-deps.svg (a *file* matching xo-*).
    rows = [r for r in rows if r.subsystem in known_subsystems]

    # 2. emptiness is judged on TOTAL loc, before any exclusion.  Judging it on
    #    code loc would drop xo-cmake, which is a real subsystem that happens to
    #    be 100% Build.
    total_by_sub = collections.Counter()
    for r in rows:
        total_by_sub[r.subsystem] += r.loc
    empty = sorted(s for s in known_subsystems if total_by_sub[s] < min_loc)
    if empty:
        notes.append("%d subsystem(s) empty, not drawn: %s"
                     % (len(empty), ", ".join(empty)))
    rows = [r for r in rows if r.subsystem not in empty]

    # 3. vendored third-party
    if not include_vendored:
        hidden = [r for r in rows if any(r.path.startswith(v) for v in vendored)]
        if hidden:
            notes.append("excluded vendored: %s (%s LOC, %d files)"
                         % (", ".join(vendored),
                            sum(r.loc for r in hidden), len(hidden)))
        rows = [r for r in rows if r not in set(hidden)] if hidden else rows

    # 4. generated
    if not include_generated:
        hidden = [r for r in rows if r.generated]
        if hidden:
            notes.append("excluded %d generated files (%s LOC)"
                         % (len(hidden), sum(r.loc for r in hidden)))
        rows = [r for r in rows if not r.generated]

    # 5. explicit kind filter
    if kinds:
        rows = [r for r in rows if r.kind in kinds]

    # 6. positional subsystem arguments restrict the OUTPUT (never the walk)
    if subsystems:
        rows = [r for r in rows if r.subsystem in set(subsystems)]

    unmapped = sorted({r.language for r in rows if r.kind == "Other"})
    if unmapped:
        notes.append("unmapped language(s) bucketed as Other: %s"
                     % ", ".join(unmapped))

    groups = {}
    for r in rows:
        cell = groups.setdefault(r.subsystem, {}).get(
            r.kind, Cell(0, 0, 0, {}))
        languages = dict(cell.languages or {})
        languages[r.language] = languages.get(r.language, 0) + r.loc
        groups[r.subsystem][r.kind] = Cell(cell.loc + r.loc,
                                           cell.complexity + r.complexity,
                                           cell.nfiles + 1,
                                           languages)

    # a subsystem can survive emptiness yet have nothing left after filtering
    drained = sorted(s for s, t in total_by_sub.items()
                     if s not in empty and s not in groups
                     and (not subsystems or s in set(subsystems)))
    if drained:
        notes.append("%d subsystem(s) have no content in this view: %s"
                     % (len(drained), ", ".join(drained)))

    return Selection(groups=groups, notes=notes)


def subsystem_loc(groups):
    """subsystem -> total LOC across its kinds."""
    return {s: sum(c.loc for c in kinds.values()) for s, kinds in groups.items()}
```

Step 3 uses a list-membership test against a `set` of `Row` namedtuples, which
is safe because `Row` is hashable (all fields are `str`/`bool`/`int`).

- [ ] **Step 4: Run tests to verify they pass**

```bash
python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v
```

Expected: PASS, all `TestSelect` and `TestSubsystemLoc` cases green.

- [ ] **Step 5: Commit**

```bash
git add xo-cmake/bin/xo-loc.in xo-cmake/utest/test_xo_loc.py
git commit -m "xo-loc: exclusions, aggregation and the honesty report [FEATURE]"
```

## Comments
