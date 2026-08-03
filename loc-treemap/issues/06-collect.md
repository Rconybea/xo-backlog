# 06 — `collect`: root discovery and the scc invocation

Status: open
Type: task

The one stage that touches the outside world. Two hazards it exists to contain:

1. **The walk must start at the repo root.** `scc` only honours `.gitignore`
   files at or below its walk root, so walking a subdirectory re-admits ~28k LOC
   of `.build` artifacts and invents phantom languages (CMake depfiles detected
   as "D", `CMakeCache.txt` as "Plain Text").
2. **The generated marker is coupled to `xo-facet/codegen/*.j2` wording.** If
   those templates are reflowed, generated-file exclusion silently stops working.
   This ticket adds a guard test that fails loudly instead.

**Files:**
- Modify: `xo-cmake/bin/xo-loc.in`
- Modify: `xo-cmake/utest/test_xo_loc.py`

**Interfaces:**
- Consumes: `normalize` (ticket 02)
- Produces:
  - `SCC_GENERATED_MARKER = "Generated automagically"`
  - `class XoLocError(Exception)`
  - `find_root(explicit=None, env=None, cwd=None) -> str`
  - `known_subsystems(root: str) -> set[str]`
  - `run_scc(root: str, scc: str = "scc") -> list` (raw parsed JSON)

---

- [ ] **Step 1: Write the failing tests**

Append to `xo-cmake/utest/test_xo_loc.py`:

```python
import os
import shutil
import subprocess
import tempfile


class TestFindRoot(unittest.TestCase):
    def setUp(self):
        self.xo_loc = load_xo_loc()
        self.tmp = tempfile.mkdtemp()
        self.addCleanup(shutil.rmtree, self.tmp)
        os.makedirs(os.path.join(self.tmp, "xo-cmake"))
        os.makedirs(os.path.join(self.tmp, "xo-gc", "src", "gc"))

    def test_explicit_root_wins(self):
        self.assertEqual(self.xo_loc.find_root(explicit=self.tmp), self.tmp)

    def test_env_is_used_when_no_explicit_root(self):
        self.assertEqual(self.xo_loc.find_root(env=self.tmp), self.tmp)

    def test_searches_upward_from_cwd(self):
        deep = os.path.join(self.tmp, "xo-gc", "src", "gc")
        self.assertEqual(self.xo_loc.find_root(cwd=deep), self.tmp)

    def test_raises_when_no_root_can_be_found(self):
        empty = tempfile.mkdtemp()
        self.addCleanup(shutil.rmtree, empty)
        with self.assertRaises(self.xo_loc.XoLocError):
            self.xo_loc.find_root(cwd=empty)


class TestKnownSubsystems(unittest.TestCase):
    def setUp(self):
        self.xo_loc = load_xo_loc()
        self.tmp = tempfile.mkdtemp()
        self.addCleanup(shutil.rmtree, self.tmp)
        os.makedirs(os.path.join(self.tmp, "xo-gc"))
        os.makedirs(os.path.join(self.tmp, "xo-cmake"))
        os.makedirs(os.path.join(self.tmp, "pkgs"))
        open(os.path.join(self.tmp, "xo-deps.svg"), "w").close()

    def test_directories_matching_xo_star_only(self):
        found = self.xo_loc.known_subsystems(self.tmp)
        self.assertEqual(found, {"xo-gc", "xo-cmake"})

    def test_excludes_a_FILE_matching_the_glob(self):
        # xo-deps.svg is a real file in this repo root and matches xo-*
        self.assertNotIn("xo-deps.svg", self.xo_loc.known_subsystems(self.tmp))


class TestRunScc(unittest.TestCase):
    def setUp(self):
        self.xo_loc = load_xo_loc()

    def test_missing_scc_raises_a_helpful_error(self):
        with self.assertRaises(self.xo_loc.XoLocError) as ctx:
            self.xo_loc.run_scc(".", scc="definitely-not-a-real-binary")
        self.assertIn("scc", str(ctx.exception).lower())


REPO_ROOT = _HERE.parent.parent


@unittest.skipUnless(shutil.which("scc"), "scc not installed")
class TestSccIntegration(unittest.TestCase):
    def setUp(self):
        self.xo_loc = load_xo_loc()

    def test_generated_marker_still_matches_the_tree(self):
        # GUARD: the marker is coupled to xo-facet/codegen/*.j2 wording.  If the
        # templates are reflowed this fails loudly rather than silently
        # counting generated code as hand-written.
        raw = self.xo_loc.run_scc(str(REPO_ROOT))
        rows = self.xo_loc.normalize(raw)
        generated = [r for r in rows if r.generated]
        self.assertGreater(
            len(generated), 100,
            "the %r marker matched %d files; has xo-facet/codegen/*.j2 changed?"
            % (self.xo_loc.SCC_GENERATED_MARKER, len(generated)))

    def test_root_walk_excludes_build_artifacts(self):
        # walking a subdirectory instead would re-admit these
        raw = self.xo_loc.run_scc(str(REPO_ROOT))
        rows = self.xo_loc.normalize(raw)
        self.assertEqual([r.path for r in rows if "/.build" in r.path], [])
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v
```

Expected: FAIL — `AttributeError: module 'xo_loc' has no attribute 'XoLocError'`.

- [ ] **Step 3: Write minimal implementation**

Add to `xo-cmake/bin/xo-loc.in` (extend the imports at the top with
`import json`, `import os`, `import subprocess`):

```python
# ----------------------------------------------------------------
# collect: the only stage that touches the outside world.

SCC_GENERATED_MARKER = "Generated automagically"


class XoLocError(Exception):
    """A condition worth reporting to the user rather than a traceback."""


def find_root(explicit=None, env=None, cwd=None):
    """Locate the umbrella source root.

    The scc walk MUST start here: scc only honours .gitignore files at or below
    its walk root, so walking a subsystem directory re-admits the .build tree.

    Ladder: --root, then $XO_SOURCE_ROOT, then git, then an upward search.
    """
    if explicit:
        return explicit
    if env:
        return env

    start = cwd or os.getcwd()

    try:
        out = subprocess.run(["git", "-C", start, "rev-parse", "--show-toplevel"],
                             stdout=subprocess.PIPE, stderr=subprocess.DEVNULL,
                             check=True, text=True)
        top = out.stdout.strip()
        if top and os.path.isdir(os.path.join(top, "xo-cmake")):
            return top
    except (OSError, subprocess.CalledProcessError):
        pass

    path = os.path.abspath(start)
    while True:
        if os.path.isdir(os.path.join(path, "xo-cmake")):
            return path
        parent = os.path.dirname(path)
        if parent == path:
            raise XoLocError(
                "cannot locate the xo source root from %s; pass --root=DIR "
                "or set $XO_SOURCE_ROOT" % start)
        path = parent


def known_subsystems(root):
    """The xo-* DIRECTORIES under `root`.

    Directories only: the root also holds xo-deps.svg, a file matching xo-*.
    """
    return {name for name in os.listdir(root)
            if name.startswith("xo-")
            and os.path.isdir(os.path.join(root, name))}


def run_scc(root, scc="scc"):
    """Run scc once over the whole tree; return parsed JSON."""
    cmd = [scc, "--by-file", "--gen",
           "--generated-markers", SCC_GENERATED_MARKER,
           "-f", "json", root]
    try:
        out = subprocess.run(cmd, stdout=subprocess.PIPE,
                             stderr=subprocess.PIPE, check=True, text=True)
    except FileNotFoundError:
        raise XoLocError(
            "%s not found on PATH.  xo-loc needs scc >= 3.5.0 to count the "
            "tree; install it into the dev environment (see shells.nix)." % scc)
    except subprocess.CalledProcessError as exc:
        raise XoLocError("%s failed: %s" % (scc, exc.stderr.strip()))

    try:
        return json.loads(out.stdout)
    except ValueError as exc:
        raise XoLocError("could not parse %s json output: %s" % (scc, exc))
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python3 -m unittest discover -s xo-cmake/utest -p 'test_*.py' -v
```

Expected: PASS. The `TestSccIntegration` cases run here because `scc` 3.5.0 is
on PATH; they skip cleanly where it is not.

- [ ] **Step 5: Commit**

```bash
git add xo-cmake/bin/xo-loc.in xo-cmake/utest/test_xo_loc.py
git commit -m "xo-loc: root discovery + scc invocation, with marker guard [FEATURE]"
```

## Comments
