# 09 — HTML renderer: self-contained, hover + click-to-zoom

Status: open
Type: task

Mirrors `xo-deps --format=html`: one self-contained file, no external assets, no
CDN. The SVG from ticket 08 is embedded verbatim and wrapped in inlined vanilla
JS. This is where the nested layout's long tail becomes readable — hover gives
you the numbers a 30×26 px tile cannot show.

**Files:**
- Modify: `xo-cmake/bin/xo-loc.in`
- Modify: `xo-cmake/utest/test_xo_loc.py`

**Interfaces:**
- Consumes: `emit_svg` (ticket 08); `RENDERERS` (ticket 07)
- Produces: `emit_html(tiles, groups, notes, args) -> str`, registered as `RENDERERS["html"]`

---

- [ ] **Step 1: Write the failing tests**

Append to `xo-cmake/utest/test_xo_loc.py`:

```python
class TestHtml(unittest.TestCase):
    def setUp(self):
        self.xo_loc = load_xo_loc()
        C = self.xo_loc.Cell
        self.groups = {
            "xo-reader2": {"Code": C(11000, 620, 200), "Build": C(1377, 40, 20)},
            "xo-gc": {"Code": C(6000, 650, 78)},
        }
        canvas = self.xo_loc.Rect(0.0, 0.0, 1200.0, 800.0)
        self.tiles, self.notes = self.xo_loc.layout(self.groups, canvas)
        self.args = self.xo_loc.build_parser().parse_args([])
        self.html = self.xo_loc.emit_html(self.tiles, self.groups,
                                          self.notes, self.args)

    def test_is_a_complete_document(self):
        self.assertTrue(self.html.lstrip().startswith("<!doctype html>"))
        self.assertIn("</html>", self.html)

    def test_embeds_the_svg(self):
        self.assertIn("<svg", self.html)
        self.assertIn("xo-reader2", self.html)

    def test_is_self_contained(self):
        # no CDN, no external stylesheet, no remote font
        for forbidden in ("http://", "https://cdn", "<link", "src=\"http"):
            self.assertNotIn(forbidden, self.html.replace(
                "http://www.w3.org/2000/svg", ""))

    def test_carries_the_exclusion_report_visibly(self):
        notes = ["excluded 369 generated files (10968 LOC)"]
        html = self.xo_loc.emit_html(self.tiles, self.groups, notes, self.args)
        # the report must be visible in the page, not just an svg comment
        self.assertIn("excluded 369 generated files", html)
        self.assertIn('class="notes"', html)

    def test_ships_a_legend(self):
        self.assertIn('class="legend"', self.html)
        self.assertIn("Build", self.html)

    def test_handles_both_color_schemes(self):
        self.assertIn("prefers-color-scheme", self.html)

    def test_has_the_interaction_hooks(self):
        self.assertIn("addEventListener", self.html)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v
```

Expected: FAIL — `AttributeError: module 'xo_loc' has no attribute 'emit_html'`.

- [ ] **Step 3: Write minimal implementation**

Add to `xo-cmake/bin/xo-loc.in`:

```python
# ----------------------------------------------------------------
# render: html.  Self-contained -- no CDN, no external assets, like
# xo-deps --format=html.

_HTML_TEMPLATE = """<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>xo-loc &mdash; %(title)s</title>
<style>
  :root { --bg: #ffffff; --fg: #1a1d21; --muted: #5a6068; --panel: #f2f4f7; }
  @media (prefers-color-scheme: dark) {
    :root { --bg: #14171a; --fg: #e6e9ed; --muted: #9aa2ac; --panel: #1f2429; }
  }
  body { margin: 0; padding: 1rem; background: var(--bg); color: var(--fg);
         font-family: system-ui, sans-serif; }
  h1 { font-size: 1rem; font-weight: 600; margin: 0 0 .5rem; }
  .wrap { overflow-x: auto; }
  svg { max-width: 100%%; height: auto; }
  .legend { display: flex; flex-wrap: wrap; gap: .75rem; margin: .5rem 0; }
  .legend span { display: inline-flex; align-items: center; gap: .35rem;
                 font-size: .8rem; color: var(--muted); }
  .swatch { width: .8rem; height: .8rem; border-radius: 2px; }
  .notes { background: var(--panel); border-radius: 4px; padding: .5rem .75rem;
           font-size: .78rem; color: var(--muted); margin-top: .75rem; }
  .notes li { margin: .15rem 0; }
  #tip { position: fixed; pointer-events: none; background: var(--panel);
         color: var(--fg); border-radius: 4px; padding: .35rem .5rem;
         font-size: .78rem; opacity: 0; transition: opacity .1s; }
  rect.dim { opacity: .25; }
</style>
</head>
<body>
<h1>%(title)s &mdash; %(total)s LOC across %(count)s subsystems</h1>
<div class="legend">%(legend)s</div>
<div class="wrap">%(svg)s</div>
<div class="notes"><ul>%(notes)s</ul></div>
<div id="tip"></div>
<script>
(function () {
  var tip = document.getElementById('tip');
  var svg = document.querySelector('svg');
  if (!svg) { return; }
  var rects = Array.prototype.slice.call(svg.querySelectorAll('rect'));
  var tiles = rects.slice(1);   // rects[0] is the background

  tiles.forEach(function (r) {
    var title = r.querySelector('title');
    var label = title ? title.textContent : '';

    r.addEventListener('mousemove', function (e) {
      tip.textContent = label;
      tip.style.opacity = '1';
      tip.style.left = (e.clientX + 12) + 'px';
      tip.style.top = (e.clientY + 12) + 'px';
    });
    r.addEventListener('mouseleave', function () {
      tip.style.opacity = '0';
    });
    r.addEventListener('click', function (e) {
      e.stopPropagation();
      var on = r.classList.contains('focus');
      tiles.forEach(function (o) {
        o.classList.remove('focus');
        o.classList.toggle('dim', !on && o !== r);
      });
      if (!on) { r.classList.add('focus'); }
    });
  });

  document.body.addEventListener('click', function () {
    tiles.forEach(function (o) {
      o.classList.remove('focus');
      o.classList.remove('dim');
    });
  });
}());
</script>
</body>
</html>
"""


def emit_html(tiles, groups, notes, args):
    svg = emit_svg(tiles, groups, [], args)

    if args.color == "complexity":
        legend = "".join(
            '<span><i class="swatch" style="background:%s"></i>%s</span>'
            % (shade, label)
            for shade, label in zip(CX_RAMP,
                                    ["low cx", "", "", "", "", "high cx"])
            if label)
        legend += ('<span><i class="swatch" style="background:%s"></i>'
                   "non-code</span>" % NEUTRAL_COLOR)
    else:
        legend = "".join(
            '<span><i class="swatch" style="background:%s"></i>%s</span>'
            % (KIND_COLORS[k], k)
            for k in ["Code", "Build", "Docs", "Data", "Generated", "Other"])

    total = sum(c.loc for kinds in groups.values() for c in kinds.values())
    title = ", ".join(args.subsystems) if args.subsystems else "xo subsystems"

    return _HTML_TEMPLATE % {
        "title": svg_escape(title),
        "total": "{:,}".format(total),
        "count": len(groups),
        "legend": legend,
        "svg": svg,
        "notes": "".join("<li>%s</li>" % svg_escape(n) for n in notes)
                 or "<li>nothing excluded</li>",
    }


RENDERERS["html"] = emit_html
```

Note the `%%` in the CSS `max-width: 100%%` — the template is applied with `%`
formatting, so a literal percent must be doubled.

- [ ] **Step 4: Run tests to verify they pass**

```bash
python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v
```

Expected: PASS.

- [ ] **Step 5: Open it and interact**

```bash
python3 xo-cmake/bin/xo-loc.in --format=html --output=/tmp/xo-loc.html
```

Open `/tmp/xo-loc.html`. Verify: hovering a tile shows LOC/complexity/file count,
clicking dims the rest, clicking the background resets, the exclusion report is
visible at the bottom, and the page is readable in both light and dark mode.

- [ ] **Step 6: Commit**

```bash
git add xo-cmake/bin/xo-loc.in xo-cmake/utest/test_xo_loc.py
git commit -m "xo-loc: self-contained interactive html output [FEATURE]"
```

## Comments
