# 08 — SVG renderer

Status: open
Type: task

**Files:**
- Modify: `xo-cmake/bin/xo-loc.in`
- Modify: `xo-cmake/utest/test_xo_loc.py`

**Interfaces:**
- Consumes: `Tile`, `layout` (ticket 05); `RENDERERS` (ticket 07)
- Produces:
  - `KIND_COLORS: dict[str, str]`
  - `NEUTRAL_COLOR: str`
  - `tile_color(kind: str, cx_per_kloc: float, mode: str) -> str`
  - `emit_svg(tiles, groups, notes, args) -> str`, registered as `RENDERERS["svg"]`

---

- [ ] **Step 1: Load the dataviz skill before choosing any color**

Invoke the `dataviz` skill and follow it for the palette. The requirements it
must satisfy here:

- **6 categorical hues**, one per kind: `Code`, `Build`, `Docs`, `Data`,
  `Generated`, `Other`. `Code` should read as the primary/most-saturated hue
  since it dominates the map; `Build` and `Generated` should read as clearly
  secondary, because the whole point of the kind dimension is that build glue
  and generated code are *visibly not* hand-written code.
- **A sequential scale** for `--color=complexity` over cx/kLOC, whose observed
  range on this tree is roughly **10 → 165**.
- Adjacent tiles must stay distinguishable, and tile labels must clear contrast
  minimums against their own fill.

The palette below is a working brand-neutral starting point so this ticket is
executable standalone. **Replace its values with the dataviz-validated palette**;
keep the key names.

- [ ] **Step 2: Write the failing tests**

Append to `xo-cmake/utest/test_xo_loc.py`:

```python
import xml.etree.ElementTree as ET


class TestSvg(unittest.TestCase):
    def setUp(self):
        self.xo_loc = load_xo_loc()
        C = self.xo_loc.Cell
        self.groups = {
            "xo-reader2": {"Code": C(11000, 620, 200), "Build": C(1377, 40, 20)},
            "xo-gc": {"Code": C(6000, 650, 78)},
            "xo-symboltable": {"Code": C(76, 3, 4), "Build": C(33, 1, 2)},
        }
        canvas = self.xo_loc.Rect(0.0, 0.0, 1200.0, 800.0)
        self.tiles, self.notes = self.xo_loc.layout(self.groups, canvas)
        self.args = self.xo_loc.build_parser().parse_args([])

    def render(self, **kw):
        for k, v in kw.items():
            setattr(self.args, k, v)
        return self.xo_loc.emit_svg(self.tiles, self.groups, self.notes, self.args)

    def test_output_is_well_formed_xml(self):
        root = ET.fromstring(self.render())
        self.assertTrue(root.tag.endswith("svg"))

    def test_declares_the_canvas_size(self):
        root = ET.fromstring(self.render())
        self.assertEqual(root.get("width"), "1200")
        self.assertEqual(root.get("height"), "800")

    def test_sets_an_explicit_background(self):
        # xo-deps.in:196 records that inheriting the viewer's background
        # renders elements near-invisible
        root = ET.fromstring(self.render())
        rects = root.iter("{http://www.w3.org/2000/svg}rect")
        first = next(rects)
        self.assertEqual(first.get("width"), "1200")
        self.assertEqual(first.get("height"), "800")
        self.assertTrue(first.get("fill"))

    def test_draws_a_rect_per_leaf(self):
        root = ET.fromstring(self.render())
        rects = list(root.iter("{http://www.w3.org/2000/svg}rect"))
        # At the production A_MIN this fixture floors nothing (F = 22.3), so:
        #   background 1
        #   xo-reader2      2 kinds, 642752 px^2 -> subdivided  -> 2
        #   xo-gc           1 kind                -> flat        -> 1
        #   xo-symboltable  2 kinds,   5660 px^2 -> subdivided  -> 2
        self.assertEqual(len(rects), 1 + 2 + 1 + 2)

    def test_single_kind_subsystem_is_never_subdivided(self):
        root = ET.fromstring(self.render())
        titles = [t.text for t in root.iter("{http://www.w3.org/2000/svg}title")]
        self.assertEqual([t for t in titles if t.startswith("xo-gc")],
                         [t for t in titles if t.startswith("xo-gc:")])

    def test_labels_the_subsystems(self):
        text = self.render()
        self.assertIn("xo-reader2", text)
        self.assertIn("xo-gc", text)

    def test_tooltip_carries_everything_the_spec_requires(self):
        C = self.xo_loc.Cell
        cell = C(490, 49, 2, {"C++": 400, "C++ Header": 90})
        tip = self.xo_loc._tooltip("xo-gc", "Code", cell)
        self.assertIn("490 LOC", tip)      # loc
        self.assertIn("cx 49", tip)        # complexity
        self.assertIn("100/kLOC", tip)     # cx per kloc
        self.assertIn("2 files", tip)      # file count
        self.assertIn("C++ 400", tip)      # language breakdown
        self.assertIn("C++ Header 90", tip)

    def test_escapes_markup_in_labels(self):
        self.assertEqual(self.xo_loc.svg_escape("a&b<c>"), "a&amp;b&lt;c&gt;")

    def test_kind_mode_colors_by_kind(self):
        self.assertEqual(
            self.xo_loc.tile_color("Build", 0.0, "kind"),
            self.xo_loc.KIND_COLORS["Build"])

    def test_complexity_mode_greys_out_non_code(self):
        self.assertEqual(self.xo_loc.tile_color("Docs", 0.0, "complexity"),
                         self.xo_loc.NEUTRAL_COLOR)
        self.assertNotEqual(self.xo_loc.tile_color("Code", 120.0, "complexity"),
                            self.xo_loc.NEUTRAL_COLOR)

    def test_complexity_scale_is_monotonic(self):
        low = self.xo_loc.tile_color("Code", 20.0, "complexity")
        high = self.xo_loc.tile_color("Code", 160.0, "complexity")
        self.assertNotEqual(low, high)
```

- [ ] **Step 3: Run tests to verify they fail**

```bash
python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v
```

Expected: FAIL — `AttributeError: module 'xo_loc' has no attribute 'emit_svg'`.

- [ ] **Step 4: Write minimal implementation**

Add to `xo-cmake/bin/xo-loc.in`:

```python
# ----------------------------------------------------------------
# render: svg

# Starting palette -- replace with the dataviz-validated values, keep the keys.
KIND_COLORS = {
    "Code":      "#3b6ea5",
    "Build":     "#8a8f98",
    "Docs":      "#5a9e6f",
    "Data":      "#b5893d",
    "Generated": "#7a6ba8",
    "Other":     "#a85a5a",
}
NEUTRAL_COLOR = "#d4d7dc"
BACKGROUND_COLOR = "#ffffff"
LABEL_COLOR = "#ffffff"

# observed cx/kLOC range on this tree
CX_LO = 10.0
CX_HI = 165.0

# light -> dark ramp for --color=complexity
CX_RAMP = ["#dbe7f3", "#b7cde7", "#8fb0d6", "#6790c2", "#426fa8", "#28507f"]


def svg_escape(text):
    return (str(text).replace("&", "&amp;")
                     .replace("<", "&lt;")
                     .replace(">", "&gt;"))


def tile_color(kind, cx_per_kloc, mode):
    if mode == "complexity":
        # scc complexity is ~0 for Docs/Data/Build languages, so a complexity
        # scale is only meaningful over Code
        if kind != "Code":
            return NEUTRAL_COLOR
        span = CX_HI - CX_LO
        t = 0.0 if span <= 0 else (cx_per_kloc - CX_LO) / span
        t = min(1.0, max(0.0, t))
        return CX_RAMP[min(len(CX_RAMP) - 1, int(t * len(CX_RAMP)))]
    return KIND_COLORS.get(kind, KIND_COLORS["Other"])


def _cx_per_kloc(cell):
    return (1000.0 * cell.complexity / cell.loc) if cell.loc else 0.0


def _languages_of(cells):
    """Merged 'C++ 400, CMake 25' summary, biggest first."""
    merged = {}
    for cell in cells:
        for language, loc in (cell.languages or {}).items():
            merged[language] = merged.get(language, 0) + loc
    ranked = sorted(merged.items(), key=lambda kv: (-kv[1], kv[0]))
    return ", ".join("%s %d" % (language, loc) for language, loc in ranked)


def _tooltip(subsystem, kind, cell):
    """The spec requires LOC, complexity, cx/kLOC, file count and languages."""
    return ("%s / %s: %d LOC, cx %d (%.0f/kLOC), %d files [%s]"
            % (subsystem, kind, cell.loc, cell.complexity,
               _cx_per_kloc(cell), cell.nfiles, _languages_of([cell])))


def _svg_rect(rect, fill, title):
    return ('  <rect x="%.2f" y="%.2f" width="%.2f" height="%.2f" '
            'fill="%s" stroke="%s" stroke-width="1"><title>%s</title></rect>'
            % (rect.x, rect.y, rect.w, rect.h, fill, BACKGROUND_COLOR,
               svg_escape(title)))


def _svg_label(rect, text):
    if rect.w < LABEL_W or rect.h < LABEL_H:
        return None
    return ('  <text x="%.2f" y="%.2f" font-family="sans-serif" '
            'font-size="11" fill="%s">%s</text>'
            % (rect.x + 4, rect.y + 13, LABEL_COLOR, svg_escape(text)))


def emit_svg(tiles, groups, notes, args):
    out = [
        '<svg xmlns="http://www.w3.org/2000/svg" width="%d" height="%d" '
        'viewBox="0 0 %d %d">' % (args.width, args.height, args.width, args.height),
        '  <rect x="0" y="0" width="%d" height="%d" fill="%s"/>'
        % (args.width, args.height, BACKGROUND_COLOR),
    ]

    for tile in tiles:
        kinds = groups[tile.subsystem]
        total = sum(c.loc for c in kinds.values())

        if tile.kind_rects:
            for placement in tile.kind_rects:
                cell = kinds[placement.key]
                out.append(_svg_rect(
                    placement.rect,
                    tile_color(placement.key, _cx_per_kloc(cell), args.color),
                    _tooltip(tile.subsystem, placement.key, cell)))
        else:
            # flat tile: the tooltip must still account for the whole subsystem,
            # not just its dominant kind
            cell = kinds[tile.dominant]
            complexity = sum(c.complexity for c in kinds.values())
            nfiles = sum(c.nfiles for c in kinds.values())
            out.append(_svg_rect(
                tile.rect,
                tile_color(tile.dominant, _cx_per_kloc(cell), args.color),
                "%s: %d LOC, cx %d (%.0f/kLOC), %d files, mostly %s [%s]"
                % (tile.subsystem, total, complexity,
                   1000.0 * complexity / total if total else 0.0,
                   nfiles, tile.dominant, _languages_of(kinds.values()))))

        label = _svg_label(tile.rect, tile.subsystem)
        if label:
            out.append(label)

    for note in notes:
        out.append("  <!-- %s -->" % svg_escape(note))
    out.append("</svg>")
    return "\n".join(out) + "\n"


RENDERERS["svg"] = emit_svg
```

Place `RENDERERS["svg"] = emit_svg` after the `RENDERERS` dict from ticket 07.

- [ ] **Step 5: Run tests to verify they pass**

```bash
python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v
```

Expected: PASS.

- [ ] **Step 6: Look at the actual output**

```bash
python3 xo-cmake/bin/xo-loc.in --output=/tmp/xo-loc.svg
python3 xo-cmake/bin/xo-loc.in --color=complexity --output=/tmp/xo-loc-cx.svg
```

Open both. Check by eye: no unlabelled slivers, `xo-cmake` reads as Build,
the `xo-py*` bindings read as mostly Build, and the complexity map is not
uniformly one shade.

- [ ] **Step 7: Commit**

```bash
git add xo-cmake/bin/xo-loc.in xo-cmake/utest/test_xo_loc.py
git commit -m "xo-loc: svg renderer [FEATURE]"
```

## Comments
