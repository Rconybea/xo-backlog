# 03 — ten subsystems are never built by the nix CI

Status: diagnosed (2026-09-13)
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

- [ ] `pkgs/*.nix` + `xo.nix` entries exist for the seven with no derivation
- [ ] `ci.yaml`'s step list covers every subsystem `ci.nix` exports
- [ ] `ci-cmake.yaml` regenerated so `xo-reflectable2` is built
- [ ] the membership check below prints nothing, and something runs it —
      a hand-maintained list that nobody diffs against `subsystem-list` is what
      produced this ticket

Progress: `grep -oP 'nix-build ci.nix -A \K[a-z0-9-]+' .forgejo/workflows/ci.yaml | sort -u > /tmp/ci; comm -13 /tmp/ci <(grep '^xo-' xo-cmake/etc/xo/subsystem-list | sort) | wc -l`

## Notes

The obvious fix for the drift is to generate `ci.yaml` the way `ci-cmake.yaml`
is generated — same `subsystem-list`, same `xo-gen-ci` target. That is more than
this ticket needs to claim, though: the nix workflow's steps carry per-subsystem
gc-root handling that the cmake one does not, so whether the template can
express them is unverified. Worth settling before hand-adding ten more steps.
