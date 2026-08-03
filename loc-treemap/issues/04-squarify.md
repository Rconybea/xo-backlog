# 04 — `squarify`: the pure layout function

Status: open
Type: task

The squarified treemap algorithm (Bruls, Huizing & van Wijk). Pure — takes
weights and a rectangle, returns rectangles. It knows nothing about SVG, kinds,
or subsystems, which is exactly what makes it testable against invariants.

**Files:**
- Modify: `xo-cmake/bin/xo-loc.in`
- Modify: `xo-cmake/utest/test_xo_loc.py`

**Interfaces:**
- Consumes: nothing (pure)
- Produces:
  - `Rect = namedtuple("Rect", "x y w h")`
  - `Placement = namedtuple("Placement", "key rect")`
  - `squarify(items: list[tuple[Any, float]], rect: Rect) -> list[Placement]`

---

- [ ] **Step 1: Write the failing tests**

Append to `xo-cmake/utest/test_xo_loc.py`:

```python
class TestSquarify(unittest.TestCase):
    def setUp(self):
        self.xo_loc = load_xo_loc()
        self.canvas = self.xo_loc.Rect(0.0, 0.0, 1200.0, 800.0)

    def area(self, rect):
        return rect.w * rect.h

    def test_empty_input(self):
        self.assertEqual(self.xo_loc.squarify([], self.canvas), [])

    def test_single_item_fills_the_rect(self):
        [p] = self.xo_loc.squarify([("a", 10)], self.canvas)
        self.assertEqual(p.key, "a")
        self.assertAlmostEqual(p.rect.w, 1200.0)
        self.assertAlmostEqual(p.rect.h, 800.0)

    def test_tiles_the_rect_exactly(self):
        items = [("s%d" % i, w) for i, w in
                 enumerate([12377, 11011, 7184, 6143, 6075, 5519, 109, 130])]
        placed = self.xo_loc.squarify(items, self.canvas)
        self.assertEqual(len(placed), len(items))
        self.assertAlmostEqual(sum(self.area(p.rect) for p in placed),
                               self.area(self.canvas), places=3)

    def test_area_is_proportional_to_weight(self):
        items = [("a", 3), ("b", 1)]
        placed = {p.key: p.rect for p in self.xo_loc.squarify(items, self.canvas)}
        self.assertAlmostEqual(self.area(placed["a"]) / self.area(placed["b"]),
                               3.0, places=6)

    def test_every_tile_stays_inside_the_canvas(self):
        items = [("s%d" % i, i + 1) for i in range(30)]
        for p in self.xo_loc.squarify(items, self.canvas):
            self.assertGreaterEqual(p.rect.x, -1e-9)
            self.assertGreaterEqual(p.rect.y, -1e-9)
            self.assertLessEqual(p.rect.x + p.rect.w, 1200.0 + 1e-9)
            self.assertLessEqual(p.rect.y + p.rect.h, 800.0 + 1e-9)

    def test_tiles_do_not_overlap(self):
        items = [("s%d" % i, i + 1) for i in range(12)]
        placed = [p.rect for p in self.xo_loc.squarify(items, self.canvas)]
        for i, a in enumerate(placed):
            for b in placed[i + 1:]:
                separated = (a.x + a.w <= b.x + 1e-6 or b.x + b.w <= a.x + 1e-6
                             or a.y + a.h <= b.y + 1e-6 or b.y + b.h <= a.y + 1e-6)
                self.assertTrue(separated, "tiles overlap: %r %r" % (a, b))

    def test_aspect_ratios_are_reasonable(self):
        # the whole point of *squarified* treemaps: no 1000:1 slivers
        items = [("s%d" % i, w) for i, w in
                 enumerate([12377, 11011, 7184, 6143, 6075, 5519, 5002, 4451])]
        for p in self.xo_loc.squarify(items, self.canvas):
            ratio = max(p.rect.w / p.rect.h, p.rect.h / p.rect.w)
            self.assertLess(ratio, 8.0, "sliver: %r" % (p.rect,))

    def test_non_positive_weights_are_skipped(self):
        placed = self.xo_loc.squarify([("a", 10), ("b", 0), ("c", -1)], self.canvas)
        self.assertEqual([p.key for p in placed], ["a"])

    def test_degenerate_rect_yields_nothing(self):
        flat = self.xo_loc.Rect(0.0, 0.0, 0.0, 800.0)
        self.assertEqual(self.xo_loc.squarify([("a", 10)], flat), [])
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v
```

Expected: FAIL — `AttributeError: module 'xo_loc' has no attribute 'Rect'`.

- [ ] **Step 3: Write minimal implementation**

Add to `xo-cmake/bin/xo-loc.in`:

```python
# ----------------------------------------------------------------
# layout: squarified treemap (Bruls, Huizing & van Wijk).
#
# Pure: weights and a rectangle in, rectangles out.  Nesting is just calling
# this again on a tile's own rect.

Rect = collections.namedtuple("Rect", "x y w h")
Placement = collections.namedtuple("Placement", "key rect")


def _worst(areas, length):
    """Worst aspect ratio in a row of the given areas laid along `length`."""
    total = sum(areas)
    if total <= 0 or length <= 0:
        return float("inf")
    hi, lo = max(areas), min(areas)
    if lo <= 0:
        return float("inf")
    return max((length * length * hi) / (total * total),
               (total * total) / (length * length * lo))


def _place_row(row, rect, out):
    """Lay `row` (list of (key, area)) along the short side of `rect`.

    Returns the leftover rect.
    """
    total = sum(a for _, a in row)
    if rect.w >= rect.h:
        width = total / rect.h if rect.h else 0.0
        y = rect.y
        for key, area in row:
            h = area / width if width else 0.0
            out.append(Placement(key, Rect(rect.x, y, width, h)))
            y += h
        return Rect(rect.x + width, rect.y, rect.w - width, rect.h)

    height = total / rect.w if rect.w else 0.0
    x = rect.x
    for key, area in row:
        w = area / height if height else 0.0
        out.append(Placement(key, Rect(x, rect.y, w, height)))
        x += w
    return Rect(rect.x, rect.y + height, rect.w, rect.h - height)


def squarify(items, rect):
    """items: [(key, weight)] -> [Placement] tiling `rect` exactly."""
    items = [(k, float(w)) for k, w in items if w > 0]
    if not items or rect.w <= 0 or rect.h <= 0:
        return []

    total = sum(w for _, w in items)
    scale = (rect.w * rect.h) / total
    scaled = sorted(((k, w * scale) for k, w in items), key=lambda kv: -kv[1])

    out = []
    remaining = rect
    while scaled:
        length = min(remaining.w, remaining.h)
        row = []
        for key, area in scaled:
            candidate = [a for _, a in row] + [area]
            if row and _worst(candidate, length) > _worst([a for _, a in row], length):
                break
            row.append((key, area))
        remaining = _place_row(row, remaining, out)
        scaled = scaled[len(row):]
    return out
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v
```

Expected: PASS, all 9 `TestSquarify` cases.

- [ ] **Step 5: Commit**

```bash
git add xo-cmake/bin/xo-loc.in xo-cmake/utest/test_xo_loc.py
git commit -m "xo-loc: squarified treemap layout [FEATURE]"
```

## Comments
