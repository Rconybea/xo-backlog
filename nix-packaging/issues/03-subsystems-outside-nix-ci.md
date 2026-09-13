# 03 — ten subsystems are never built by the nix CI

Status: fixed 2026-09-13
Type: coverage gap
Raised: found while verifying `python-packaging/01`, which named a nix attribute
that turned out not to exist.

`nix-build` is the only check that exercises an installed package config the way
a real consumer would — `CONVENTIONS.md` says so, and it has caught
`-lindentlog`, the xo-tokenizer example, and xo-reader's empty
`propagatedBuildInputs`, none of which the umbrella build saw. Ten subsystems
are outside it.

## Measured 2026-09-13

```bash
grep -oP 'nix-build ci.nix -A \K[a-z0-9-]+' .forgejo/workflows/ci.yaml | sort -u > /tmp/ci
comm -13 /tmp/ci <(grep '^xo-' xo-cmake/etc/xo/subsystem-list | sort)
#   xo-callback2 xo-equable2 xo-hashable2 xo-pyarena xo-pyfacet xo-pyindentlog2
#   xo-pyobject2 xo-pyreactor2 xo-reactor2 xo-reflectable2
```

62 of 71 built. The ten split into **two different problems**, and conflating
them would send the fix to the wrong place:

```bash
HAVE=$(grep -oP '^\s+\Kxo-[a-z0-9]+(?= *= *callPackage)' xo.nix | sort -u)
MISS=$(comm -13 /tmp/ci <(grep '^xo-' xo-cmake/etc/xo/subsystem-list | sort))

comm -13 <(echo "$HAVE") <(echo "$MISS")   # no derivation at all
#   xo-callback2 xo-pyarena xo-pyfacet xo-pyindentlog2 xo-pyobject2
#   xo-pyreactor2 xo-reactor2

comm -12 <(echo "$HAVE") <(echo "$MISS")   # has one; the workflow never calls it
#   xo-equable2 xo-hashable2 xo-reflectable2
```

**Seven have no `pkgs/*.nix` and no `xo.nix` entry.** Nothing to build. Note
the shape of the seven: `xo-reactor2` + `xo-callback2` is one cluster, and
`xo-pyarena`/`xo-pyindentlog2`/`xo-pyfacet`/`xo-pyobject2`/`xo-pyreactor2` is
the entire python side of the facet stack — every python subsystem added since
the facet work began. Their C++ counterparts are all packaged
(`xo-arena`, `xo-indentlog2`, `xo-facet`, `xo-object2` are in `xo.nix`), so
this is not a levelization consequence; it is packaging that stopped being
added at a particular point in time.

**Three have a derivation and an `xo.nix` entry, and `ci.nix` exports them** —
they are simply absent from the workflow's step list.

## Why it went unnoticed

The two CI workflows are maintained differently, and only one of them can drift.

| | `.forgejo/workflows/ci.yaml` (nix) | `.forgejo/workflows/ci-cmake.yaml` |
|---|---|---|
| source | hand-written | generated from `ci-cmake.yaml.j2` |
| membership | one hand-added step per subsystem | `<% for sub in subsystems %>` over `subsystem-list` |
| regenerate | n/a | `cmake --build .build --target xo-gen-ci` |

So adding a subsystem to `subsystem-list` puts it in the cmake workflow at the
next regeneration and in the nix workflow never. The cmake side bears this out —
it misses only three, and two are deliberate:

```bash
grep -oP 'xo-build[^\n]*?\K\bxo-[a-z0-9]+' .forgejo/workflows/ci-cmake.yaml | sort -u > /tmp/cm
comm -13 /tmp/cm <(grep '^xo-' xo-cmake/etc/xo/subsystem-list | sort)
#   xo-cmake xo-imgui xo-reflectable2
```

`xo-cmake` is the bootstrap and `xo-imgui` was excluded deliberately (umbrella
`e77b339c`). `xo-reflectable2` is the one real miss, and it is a *stale
generated file*, not a missing hand edit — one `xo-gen-ci` run fixes it.

**Nothing reports this.** A workflow that builds 62 of 71 subsystems is green,
and the nine it skips look exactly like the ones it runs. This is the same
species as the two counted in `CONVENTIONS.md` — the nix `doCheck` packages that
ran zero tests and reported success, and `--all` silently covering a smaller set
than `subsystem-list` — a harness reporting green over a set nobody measured.

## Not a blocker for the python-packaging work

`python-packaging/01` moved every extension module from `lib/` to
`lib/python/xo/`. That the change is right in a from-scratch nix build was
verified against the packaged py subsystems, which is enough:

```bash
P=$(nix-build ci.nix -A xo-pyreflect --no-out-link)
find $P -name '*.so' -path '*python*'
#   .../lib/python/xo/reflect.cpython-312-x86_64-linux-gnu.so
```

But `xo-pyfacet` and `xo-pyobject2` — the two whose cross-module import and
`keep_alive` edges are the most intricate python wiring in the tree — could not
be checked that way, because they have no derivation.

## Done when

- [x] `pkgs/*.nix` + `xo.nix` entries exist for the seven with no derivation
- [x] `ci.yaml`'s step list covers every subsystem `ci.nix` exports
- [x] `ci-cmake.yaml` regenerated so `xo-reflectable2` is built
- [x] the membership check below prints nothing, and something runs it —
      a hand-maintained list that nobody diffs against `subsystem-list` is what
      produced this ticket

Progress: `grep -oP 'nix-build ci.nix -A \K[a-z0-9-]+' .forgejo/workflows/ci.yaml | sort -u > /tmp/ci; comm -13 /tmp/ci <(grep '^xo-' xo-cmake/etc/xo/subsystem-list | sort) | wc -l`

## Fixed 2026-09-13

Seven derivations written, and `ci.yaml` is now generated rather than
hand-maintained.

### Correction: the Notes below were wrong, and it changed the fix

This ticket originally said the nix workflow's steps "carry per-subsystem
gc-root handling that the cmake one does not, so whether the template can
express them is unverified", and recommended hand-adding ten steps. Measured
instead:

```bash
python3 - <<'EOF'
import re, pathlib
s = pathlib.Path(".forgejo/workflows/ci.yaml").read_text()
b = re.findall(r"      - name: build (xo-[a-z0-9-]+)\n        run: \|\n((?:          .*\n)+)", s)
print(len(b), "steps,", len({body.replace(sub,"<SUB>") for sub,body in b}), "distinct shapes")
EOF
#   61 steps, 1 distinct shape
```

The gc-root handling is entirely uniform — `$XO_GCROOTS/<sub>` — so a template
expresses it trivially. Kept here because it is the kind of claim that sounds
like it came from reading the file and did not, and because believing it would
have produced the worse fix: ten hand-added steps and the drift mechanism left
running.

`.forgejo/workflows/ci.yaml.j2` now renders from `subsystem-list` through a
third `COMMAND` in the `xo-gen-ci` target, beside the two cmake templates —
bringing the nix pipeline under the principle `CMakeLists.txt:221` already
stated for the others. No exclusions, unlike ci-cmake: this job runs on `host`
(so xo-imgui's OpenGL is present) and nix builds xo-cmake as an ordinary
derivation.

**Generation turns a silent omission into a loud failure.** `xo.nix` and
`ci.nix` are still hand-maintained, so a subsystem added to `subsystem-list`
without a derivation now produces a step that fails:

```bash
nix-build ci.nix -A xo-notasubsystem; echo $?   # 1
```

That is the real answer to the last done-when. The check does not need a
runner — the drift now breaks the build instead of shrinking the set silently.

### The hazard in converting a hand-written workflow to a generated one

The first generated `ci.yaml` **dropped `xo-docs-site`** and the entire
`publish docs` step, including its external `curl --fail` check against
https://conybeare.us/xo-docs/. `xo-docs-site` is a nix attribute with no entry
in `subsystem-list`, so a template built from "everything before the first
build step" silently discards everything after the last one. The template keeps
a head *and* a tail; the step list is the only generated part. Caught by
diffing attribute sets rather than eyeballing:

```bash
comm -23 <(grep -oP 'nix-build ci.nix -A \K[a-z0-9-]+' ci.yaml.before | sort -u) \
         <(grep -oP 'nix-build ci.nix -A \K[a-z0-9-]+' .forgejo/workflows/ci.yaml | sort -u)
#   empty -- nothing lost
```

### xo-pyobject2's 25 python tests run under nix

They import `xo.arena`, `xo.indentlog2`, `xo.facet` and `xo.object2`, but only
`xo.object2` is built in that derivation. A `preCheck` composes the rest from
sibling store paths:

```nix
preCheck = ''
  export PYTHONPATH=${xo-pyfacet}/lib/python:${xo-pyindentlog2}/lib/python:${xo-pyarena}/lib/python
'';
```

**This works only because `xo` is a PEP 420 namespace package** — each store
path contributes a portion and python merges them. With an `__init__.py` the
first would shadow the rest and the tests could not run here at all. It is the
first load-bearing use of that property (`python-packaging/01`).

Falsified rather than assumed:

```bash
nix-build --no-out-link -E '(import ./ci.nix {}).xo-pyobject2.overrideAttrs (o: { preCheck = ""; })'
#   utest.pyobject2 ...***Failed
#   ModuleNotFoundError: No module named 'xo.arena'
```

And the tests genuinely run rather than ctest finding none — `Ran 25 tests, OK`
locally, `utest.pyobject2 ... Passed 0.19 sec` in nix, the same suite.

### A grep that looked complete and was not

`xo-callback2` was the one derivation that failed to build first time:

```
utest.callback2] find_package(callback) (xo_dependency_helper)
CMake Error: Could not find a package configuration file provided by "callback"
```

Because the survey used

```bash
grep -ohP 'xo_(dependency|pybind11_dependency|pybind11_header_dependency)\(...'
```

which misses `xo_headeronly_dependency` — the form xo-callback2 uses for all
three of its library deps (`xo-callback2/CMakeLists.txt:38-40`). The pattern
that covers every form:

```bash
grep -rhoP 'xo_[a-z_0-9]*dependency\(\s*\$?\{?[A-Za-z_]*\}?\s+\K[a-zA-Z_0-9:]+' <sub> --include=CMakeLists.txt
```

Re-run across all seven it changed exactly one answer, which is why the other
six built on the first try — and why the miss was easy not to notice.

## Still open, deliberately out of scope

**22 packages have `doCheck` without `-DENABLE_TESTING=1`**, so ctest runs and
finds nothing while reporting success — the defect `CONVENTIONS.md` already
counts:

```bash
for f in pkgs/xo-*.nix; do grep -q doCheck $f || continue
  grep -q ENABLE_TESTING $f || echo -n "$(basename $f .nix) "; done
#   xo-callback xo-pydistribution xo-pyexpression xo-pyjit xo-pykalmanfilter
#   xo-pyprintjson xo-pyprocess xo-pyreactor xo-pysimulator xo-pyunit xo-pyutil
#   xo-pywebsock xo-pywebutil xo-randomgen xo-reader xo-reflectutil xo-simulator
#   xo-statistics xo-subsys xo-tokenizer xo-websock xo-webutil
```

Not touched here — the seven new derivations simply do not repeat it. Worth its
own ticket: unlike this one, fixing it will surface tests that have never run
and may not pass.
