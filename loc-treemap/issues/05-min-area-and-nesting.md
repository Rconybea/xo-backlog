# 05 — Minimum-area floor and two-level nesting

Status: open
Type: task

Two coupled rules from the spec:

- **Derived floor.** `F = T·a / (1 − N·a)`. This is not a tuned constant — it is
  solved so that the smallest tile lands at exactly `A_MIN`. The algebra:
  `W = Σ max(wᵢ, F) ≤ T + N·F`, so `F/W ≥ a` iff `F ≥ aT/(1 − aN)`. That makes
  "every tile is at least `A_MIN`" a **provable invariant**, and the tests
  assert it rather than asserting a magic number.
- **Subdivide only when it fits.** A floored tile is by construction exactly big
  enough for its own label, so subdividing it would immediately re-hide it.

**Files:**
- Modify: `xo-cmake/bin/xo-loc.in`
- Modify: `xo-cmake/utest/test_xo_loc.py`

**Interfaces:**
- Consumes: `Rect`, `Placement`, `squarify` (ticket 04); `Cell`, `subsystem_loc` (ticket 03)
- Produces:
  - `LABEL_W = 64.0`, `LABEL_H = 18.0`, `A_MIN = 1152.0`
  - `FloorResult = namedtuple("FloorResult", "weights floored inflation")`
  - `apply_min_area(weights: dict[str, int], canvas_area: float, min_tile_area: float = A_MIN) -> FloorResult`
  - `Tile = namedtuple("Tile", "subsystem rect kind_rects dominant")` where
    `kind_rects: list[Placement]` is empty when the tile is not subdivided
  - `layout(groups, canvas: Rect, min_tile_area: float = A_MIN) -> tuple[list[Tile], list[str]]`

---

- [ ] **Step 1: Write the failing tests**

Append to `xo-cmake/utest/test_xo_loc.py`:

```python
# The real default-view distribution.  This MUST be the full 62-subsystem table:
# F depends on N as well as T, so a trimmed subset changes the answer.  With all
# 62 entries F = 173.45 and exactly three subsystems are floored; with a
# 14-entry subset F drops to 86 and nothing is floored at all.
REAL_WEIGHTS = {
    "xo-reader2": 12377, "xo-imgui": 11011, "xo-unit": 7184, "xo-ordinaltree": 6143,
    "xo-gc": 6075, "xo-reader": 5519, "xo-arena": 5002, "xo-expression2": 4451,
    "xo-jit": 4387, "xo-alloc": 4359, "xo-alloc2": 4158, "xo-indentlog": 3949,
    "xo-cmake": 3767, "xo-expression": 3268, "xo-indentlog2": 3127, "xo-tokenizer": 3082,
    "xo-interpreter2": 3040, "xo-websock": 2711, "xo-object2": 2563, "xo-ppsink": 2412,
    "xo-kalmanfilter": 2285, "xo-reflect": 2281, "xo-object": 2275, "xo-ratio": 1950,
    "xo-procedure2": 1908, "xo-reactor": 1850, "xo-interpreter": 1796, "xo-facet": 1766,
    "xo-tokenizer2": 1712, "xo-flatstring": 1509, "xo-type": 1377, "xo-stringtable2": 1355,
    "xo-numeric": 1306, "xo-process": 1255, "xo-refcnt": 845, "xo-simulator": 823,
    "xo-distribution": 804, "xo-pykalmanfilter": 801, "xo-printjson": 699,
    "xo-pysimulator": 612, "xo-randomgen": 464, "xo-pyprocess": 458, "xo-pyreactor": 399,
    "xo-timeutil": 381, "xo-callback": 356, "xo-allocutil": 334, "xo-pyjit": 332,
    "xo-subsys": 305, "xo-pyexpression": 283, "xo-testutil": 270, "xo-webutil": 263,
    "xo-pyprintjson": 254, "xo-pyreflect": 244, "xo-reflectutil": 235, "xo-pywebutil": 224,
    "xo-pyunit": 214, "xo-statistics": 199, "xo-pyutil": 196, "xo-pydistribution": 181,
    "xo-pywebsock": 159, "xo-printable2": 130, "xo-symboltable": 109,
}


class TestApplyMinArea(unittest.TestCase):
    def setUp(self):
        self.xo_loc = load_xo_loc()
        self.canvas_area = 1200.0 * 800.0

    def test_no_floor_needed_leaves_weights_untouched(self):
        weights = {"a": 50000, "b": 50000}
        result = self.xo_loc.apply_min_area(weights, self.canvas_area)
        self.assertEqual(result.weights, weights)
        self.assertEqual(result.floored, [])
        self.assertAlmostEqual(result.inflation, 0.0)

    def test_floors_only_the_small_subsystems(self):
        result = self.xo_loc.apply_min_area(REAL_WEIGHTS, self.canvas_area)
        self.assertEqual(set(result.floored),
                         {"xo-symboltable", "xo-printable2", "xo-pywebsock"})

    def test_distortion_stays_tiny(self):
        result = self.xo_loc.apply_min_area(REAL_WEIGHTS, self.canvas_area)
        self.assertAlmostEqual(result.inflation, 0.000914, places=5)  # ~0.09%

    def test_INVARIANT_every_floored_tile_reaches_the_minimum_area(self):
        # this is the property F was derived to guarantee
        result = self.xo_loc.apply_min_area(REAL_WEIGHTS, self.canvas_area)
        canvas = self.xo_loc.Rect(0.0, 0.0, 1200.0, 800.0)
        placed = self.xo_loc.squarify(sorted(result.weights.items()), canvas)
        for p in placed:
            self.assertGreaterEqual(p.rect.w * p.rect.h,
                                    self.xo_loc.A_MIN - 1e-6,
                                    "%s is below the legible minimum" % p.key)

    def test_too_many_subsystems_to_floor_is_reported_not_crashed(self):
        # N * a >= 1: the canvas cannot give everyone a legible tile
        weights = {"s%d" % i: 1 for i in range(2000)}
        result = self.xo_loc.apply_min_area(weights, self.canvas_area)
        self.assertEqual(result.weights, weights)
        self.assertEqual(result.floored, [])


class TestLayout(unittest.TestCase):
    # A two-subsystem fixture is too small for the production A_MIN to floor
    # anything, so these pass an explicit min_tile_area chosen to exercise both
    # branches deterministically: at 20000 px^2 the floor is 271 (> 109), so
    # xo-symboltable is floored and lands at ~20601 px^2 -- above one sub-tile
    # but below the 2 * 20000 needed to split it in two.
    MIN_TILE = 20000.0

    def setUp(self):
        self.xo_loc = load_xo_loc()
        self.canvas = self.xo_loc.Rect(0.0, 0.0, 1200.0, 800.0)
        C = self.xo_loc.Cell
        self.groups = {
            "xo-reader2": {"Code": C(11000, 620, 200), "Build": C(1377, 40, 20)},
            "xo-symboltable": {"Code": C(76, 3, 4), "Build": C(33, 1, 2)},
        }

    def run_layout(self):
        return self.xo_loc.layout(self.groups, self.canvas,
                                  min_tile_area=self.MIN_TILE)

    def test_big_tile_is_subdivided_by_kind(self):
        tiles, _ = self.run_layout()
        big = next(t for t in tiles if t.subsystem == "xo-reader2")
        self.assertEqual({p.key for p in big.kind_rects}, {"Code", "Build"})

    def test_floored_tile_is_NOT_subdivided(self):
        # it is exactly one label big; splitting it would re-hide it
        tiles, _ = self.run_layout()
        small = next(t for t in tiles if t.subsystem == "xo-symboltable")
        self.assertEqual(small.kind_rects, [])
        self.assertEqual(small.dominant, "Code")

    def test_sub_tiles_tile_their_parent_exactly(self):
        tiles, _ = self.run_layout()
        big = next(t for t in tiles if t.subsystem == "xo-reader2")
        self.assertAlmostEqual(sum(p.rect.w * p.rect.h for p in big.kind_rects),
                               big.rect.w * big.rect.h, places=3)

    def test_reports_the_floor_it_applied(self):
        _, notes = self.run_layout()
        self.assertTrue(any("min-area floor" in n for n in notes))
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v
```

Expected: FAIL — `AttributeError: module 'xo_loc' has no attribute 'apply_min_area'`.

- [ ] **Step 3: Write minimal implementation**

Add to `xo-cmake/bin/xo-loc.in`:

```python
# a tile must be able to hold its own label
LABEL_W = 64.0
LABEL_H = 18.0
A_MIN = LABEL_W * LABEL_H

FloorResult = collections.namedtuple("FloorResult", "weights floored inflation")
Tile = collections.namedtuple("Tile", "subsystem rect kind_rects dominant")


def apply_min_area(weights, canvas_area, min_tile_area=A_MIN):
    """Raise small weights so no tile falls below `min_tile_area`.

    Solves F = T*a/(1 - N*a), which is exactly the F for which
    F / sum(max(w, F)) >= a -- i.e. the smallest tile lands at min_tile_area.
    """
    total = float(sum(weights.values()))
    n = len(weights)
    if total <= 0 or n == 0:
        return FloorResult(dict(weights), [], 0.0)

    a = min_tile_area / canvas_area
    if n * a >= 1.0:
        # the canvas is too small to give everyone a legible tile; do not
        # pretend otherwise by inflating into nonsense
        return FloorResult(dict(weights), [], 0.0)

    floor = total * a / (1.0 - n * a)
    floored = sorted(k for k, w in weights.items() if w < floor)
    if not floored:
        return FloorResult(dict(weights), [], 0.0)

    adjusted = {k: max(float(w), floor) for k, w in weights.items()}
    inflation = (sum(adjusted.values()) - total) / total
    return FloorResult(adjusted, floored, inflation)


def layout(groups, canvas, min_tile_area=A_MIN):
    """groups -> ([Tile], [note]).  Two levels: subsystem, then kind."""
    notes = []
    totals = subsystem_loc(groups)
    if not totals:
        return [], notes

    floor = apply_min_area(totals, canvas.w * canvas.h, min_tile_area)
    if floor.floored:
        notes.append("min-area floor applied to %d subsystem(s) (+%.2f%% area): %s"
                     % (len(floor.floored), 100.0 * floor.inflation,
                        ", ".join(floor.floored)))

    tiles = []
    for placement in squarify(sorted(floor.weights.items()), canvas):
        kinds = groups[placement.key]
        dominant = max(sorted(kinds.items()), key=lambda kv: kv[1].loc)[0]

        # subdivide only if every sub-tile can still hold a label
        area = placement.rect.w * placement.rect.h
        if len(kinds) > 1 and area >= len(kinds) * min_tile_area:
            kind_rects = squarify(
                sorted((k, c.loc) for k, c in kinds.items()), placement.rect)
        else:
            kind_rects = []

        tiles.append(Tile(subsystem=placement.key, rect=placement.rect,
                          kind_rects=kind_rects, dominant=dominant))
    return tiles, notes
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v
```

Expected: PASS, all `TestApplyMinArea` and `TestLayout` cases.

- [ ] **Step 5: Commit**

```bash
git add xo-cmake/bin/xo-loc.in xo-cmake/utest/test_xo_loc.py
git commit -m "xo-loc: derived min-area floor + two-level nesting [FEATURE]"
```

## Comments
