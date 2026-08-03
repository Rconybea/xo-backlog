# 07 — CLI wiring, csv/json output, and the report footer

Status: open
Type: task

First point at which the tool runs end to end. The table formats come first
because they need no layout and no palette, so they prove the whole pipeline
before any drawing code exists — the same reason `xo-deps` can emit `--format=dot`.

**Files:**
- Modify: `xo-cmake/bin/xo-loc.in`
- Modify: `xo-cmake/utest/test_xo_loc.py`

**Interfaces:**
- Consumes: everything from tickets 02–06
- Produces:
  - `build_parser() -> argparse.ArgumentParser`
  - `emit_csv(tiles, groups, notes, args) -> str`
  - `emit_json(tiles, groups, notes, args) -> str`
  - `RENDERERS: dict[str, callable]` — tickets 08 and 09 add `"svg"` and `"html"`
  - `main(argv=None) -> int`

---

- [ ] **Step 1: Write the failing tests**

Append to `xo-cmake/utest/test_xo_loc.py`:

```python
import csv as csvmod
import io


class TestParser(unittest.TestCase):
    def setUp(self):
        self.xo_loc = load_xo_loc()
        self.parser = self.xo_loc.build_parser()

    def test_defaults_match_the_spec(self):
        args = self.parser.parse_args([])
        self.assertEqual(args.format, "svg")
        self.assertEqual(args.color, "kind")
        self.assertEqual(args.min_loc, 20)
        self.assertEqual(args.width, 1200)
        self.assertEqual(args.height, 800)
        self.assertFalse(args.include_generated)
        self.assertFalse(args.include_vendored)
        self.assertEqual(args.subsystems, [])

    def test_positional_subsystems(self):
        args = self.parser.parse_args(["xo-gc", "xo-arena"])
        self.assertEqual(args.subsystems, ["xo-gc", "xo-arena"])

    def test_kinds_is_split_on_commas(self):
        args = self.parser.parse_args(["--kinds=Code,Build"])
        self.assertEqual(args.kinds, ["Code", "Build"])

    def test_listing_Generated_in_kinds_implies_include_generated(self):
        args = self.xo_loc.resolve_args(
            self.parser.parse_args(["--kinds=Code,Generated"]))
        self.assertTrue(args.include_generated)

    def test_png_is_rejected(self):
        with self.assertRaises(SystemExit):
            self.parser.parse_args(["--format=png"])


class TestTableFormats(unittest.TestCase):
    def setUp(self):
        self.xo_loc = load_xo_loc()
        C = self.xo_loc.Cell
        self.groups = {
            "xo-gc": {"Code": C(490, 49, 2), "Build": C(25, 1, 1)},
            "xo-cmake": {"Build": C(1066, 30, 1)},
        }
        canvas = self.xo_loc.Rect(0.0, 0.0, 1200.0, 800.0)
        self.tiles, _ = self.xo_loc.layout(self.groups, canvas)
        self.args = self.xo_loc.build_parser().parse_args([])

    def test_csv_has_a_row_per_subsystem_kind(self):
        text = self.xo_loc.emit_csv(self.tiles, self.groups, [], self.args)
        rows = list(csvmod.DictReader(io.StringIO(text)))
        self.assertEqual(len(rows), 3)
        gc_code = next(r for r in rows
                       if r["subsystem"] == "xo-gc" and r["kind"] == "Code")
        self.assertEqual(gc_code["loc"], "490")
        self.assertEqual(gc_code["complexity"], "49")
        self.assertEqual(gc_code["nfiles"], "2")

    def test_json_carries_totals_and_notes(self):
        text = self.xo_loc.emit_json(self.tiles, self.groups,
                                     ["excluded 369 generated files"], self.args)
        data = json.loads(text)
        self.assertEqual(data["total_loc"], 1581)
        self.assertEqual(data["subsystems"]["xo-gc"]["loc"], 515)
        self.assertEqual(data["notes"], ["excluded 369 generated files"])

    def test_json_reports_complexity_per_kloc(self):
        text = self.xo_loc.emit_json(self.tiles, self.groups, [], self.args)
        data = json.loads(text)
        # xo-gc: 50 complexity over 515 loc
        self.assertAlmostEqual(data["subsystems"]["xo-gc"]["cx_per_kloc"],
                               1000.0 * 50 / 515, places=3)


class TestMain(unittest.TestCase):
    def test_reports_a_missing_scc_without_a_traceback(self):
        xo_loc = load_xo_loc()
        err = io.StringIO()
        rc = xo_loc.main(["--scc=definitely-not-a-real-binary",
                          "--format=csv", "--output=/dev/null"], stderr=err)
        self.assertEqual(rc, 1)
        self.assertIn("scc", err.getvalue().lower())
        self.assertNotIn("Traceback", err.getvalue())
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v
```

Expected: FAIL — `AttributeError: module 'xo_loc' has no attribute 'build_parser'`.

- [ ] **Step 3: Write minimal implementation**

Add to `xo-cmake/bin/xo-loc.in` (extend imports with `import argparse`, `import csv`):

```python
# ----------------------------------------------------------------
# render: table formats.  svg/html are registered by later tickets.

def _subsystem_summary(groups):
    out = {}
    for subsystem, kinds in sorted(groups.items()):
        loc = sum(c.loc for c in kinds.values())
        complexity = sum(c.complexity for c in kinds.values())
        out[subsystem] = {
            "loc": loc,
            "complexity": complexity,
            "cx_per_kloc": (1000.0 * complexity / loc) if loc else 0.0,
            "nfiles": sum(c.nfiles for c in kinds.values()),
            "kinds": {k: {"loc": c.loc,
                          "complexity": c.complexity,
                          "nfiles": c.nfiles}
                      for k, c in sorted(kinds.items())},
        }
    return out


def emit_csv(tiles, groups, notes, args):
    buf = io.StringIO()
    writer = csv.writer(buf)
    writer.writerow(["subsystem", "kind", "loc", "complexity", "nfiles"])
    for subsystem, kinds in sorted(groups.items()):
        for kind, cell in sorted(kinds.items()):
            writer.writerow([subsystem, kind, cell.loc, cell.complexity,
                             cell.nfiles])
    return buf.getvalue()


def emit_json(tiles, groups, notes, args):
    summary = _subsystem_summary(groups)
    return json.dumps({
        "version": XO_LOC_VERSION,
        "total_loc": sum(s["loc"] for s in summary.values()),
        "subsystems": summary,
        "notes": list(notes),
    }, indent=2, sort_keys=True) + "\n"


RENDERERS = {
    "csv": emit_csv,
    "json": emit_json,
}


# ----------------------------------------------------------------
# CLI

def build_parser():
    parser = argparse.ArgumentParser(
        prog="xo-loc",
        description="Treemap of the xo subsystem world; area is lines of code.",
        epilog="Counted live with scc on every run -- there is no snapshot file.")
    parser.add_argument("subsystems", nargs="*", default=[], metavar="SUBSYSTEM",
                        help="restrict output to these subsystems")
    parser.add_argument("--format", default="svg",
                        choices=["svg", "html", "csv", "json"],
                        help="output format (default: svg)")
    parser.add_argument("--output", default=None,
                        help="output path (default: xo-loc.<format>)")
    parser.add_argument("--color", default="kind", choices=["kind", "complexity"],
                        help="what tile color encodes (default: kind)")
    parser.add_argument("--include-generated", action="store_true",
                        help="draw generated code as its own kind")
    parser.add_argument("--include-vendored", action="store_true",
                        help="draw vendored third-party code")
    parser.add_argument("--vendored", default=",".join(DEFAULT_VENDORED),
                        help="comma-separated vendored path prefixes")
    parser.add_argument("--kinds", default=None,
                        help="comma-separated kinds to include")
    parser.add_argument("--min-loc", type=int, default=20,
                        help="below this total LOC a subsystem is reported, "
                             "not drawn (default: 20)")
    parser.add_argument("--width", type=int, default=1200, help="canvas width")
    parser.add_argument("--height", type=int, default=800, help="canvas height")
    parser.add_argument("--root", default=None,
                        help="source root (default: auto-discover)")
    parser.add_argument("--scc", default="scc", help="path to the scc binary")
    parser.add_argument("--list", action="store_true",
                        help="list subsystems and exit")
    parser.add_argument("--version", action="version",
                        version="xo-loc " + XO_LOC_VERSION)
    return parser


def resolve_args(args):
    """Normalise interdependent options."""
    args.kinds = [k.strip() for k in args.kinds.split(",")] if args.kinds else None
    args.vendored = tuple(v.strip() for v in args.vendored.split(",") if v.strip())
    if args.kinds and "Generated" in args.kinds:
        args.include_generated = True
    if args.output is None:
        args.output = "xo-loc." + args.format
    return args


def main(argv=None, stdout=None, stderr=None):
    stdout = stdout or sys.stdout
    stderr = stderr or sys.stderr
    args = resolve_args(build_parser().parse_args(argv))

    try:
        root = find_root(args.root, os.environ.get("XO_SOURCE_ROOT"))
        subsystems = known_subsystems(root)

        if args.list:
            for name in sorted(subsystems):
                print(name, file=stdout)
            return 0

        rows = normalize(run_scc(root, args.scc))
        selection = select(rows, subsystems,
                           subsystems=args.subsystems,
                           include_generated=args.include_generated,
                           include_vendored=args.include_vendored,
                           vendored=args.vendored,
                           kinds=args.kinds,
                           min_loc=args.min_loc)
        if not selection.groups:
            raise XoLocError("nothing to draw after filtering")

        canvas = Rect(0.0, 0.0, float(args.width), float(args.height))
        tiles, layout_notes = layout(selection.groups, canvas)
        notes = list(selection.notes) + list(layout_notes)

        text = RENDERERS[args.format](tiles, selection.groups, notes, args)
        with open(args.output, "w") as f:
            f.write(text)

        # the map must never hide something silently
        for note in notes:
            print("xo-loc: " + note, file=stderr)
        print("xo-loc: wrote %s" % args.output, file=stderr)
        return 0

    except XoLocError as exc:
        print("xo-loc: %s" % exc, file=stderr)
        return 1


if __name__ == "__main__":
    sys.exit(main())
```

Replace the placeholder `main()` from ticket 01 with this one, and add
`import io` to the imports.

- [ ] **Step 4: Run tests to verify they pass**

```bash
python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v
```

Expected: PASS.

- [ ] **Step 5: Run the tool for real**

```bash
python3 xo-cmake/bin/xo-loc.in --format=csv --output=/tmp/xo-loc.csv
head -5 /tmp/xo-loc.csv
python3 xo-cmake/bin/xo-loc.in --format=json --output=/tmp/xo-loc.json
python3 -c "import json;d=json.load(open('/tmp/xo-loc.json'));print(d['total_loc'], len(d['subsystems']))"
```

Expected: `total_loc` ≈ **133784** across **62** subsystems, and stderr carries
the report — roughly:

```
xo-loc: 2 subsystem(s) empty, not drawn: xo-equable2, xo-hashable2
xo-loc: excluded vendored: xo-imgui/include/imgui/ (63732 LOC, ...)
xo-loc: excluded 369 generated files (10968 LOC)
xo-loc: min-area floor applied to 3 subsystem(s) (+0.09% area): xo-printable2, xo-pywebsock, xo-symboltable
```

Exact counts drift as the tree changes; the shape is what matters.

- [ ] **Step 6: Commit**

```bash
git add xo-cmake/bin/xo-loc.in xo-cmake/utest/test_xo_loc.py
git commit -m "xo-loc: CLI, csv/json output, exclusion report [FEATURE]"
```

## Comments
