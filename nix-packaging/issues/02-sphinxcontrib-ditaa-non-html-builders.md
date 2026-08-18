# 02 — sphinxcontrib-ditaa crashes every non-html sphinx builder

Status: diagnosed
Type: bug

`sphinxcontrib-ditaa` 1.0.2 calls `Builder.warn()`, removed from Sphinx in 2.0.
Any builder that is not html or latex therefore raises, and the exception
escapes as an `ExtensionError` that kills the build. We package this: roland is
the nixpkgs maintainer, so the fix can go either upstream
(github.com/sphinx-contrib/ditaa) or as a nixpkgs patch.

Measured 2026-08-18, against sphinx 8.2.3 / sphinxcontrib-ditaa 1.0.2.

## Reproduction

Two runs, same source dir, differing only in whether the *project* contains a
ditaa directive:

```bash
O=$(mktemp -d)
# xo-arena/docs -- some page in the project uses .. ditaa::
sphinx-build -b pseudoxml -q -N xo-arena/docs $O xo-arena/docs/glossary.rst; echo $?   # 2
# xo-flatstring/docs -- no ditaa anywhere in the project
sphinx-build -b pseudoxml -q -N xo-flatstring/docs $O xo-flatstring/docs/index.rst; echo $?  # 0
```

Note `glossary.rst` contains no ditaa directive of its own. Sphinx reads every
source into the environment, and `doctree-read` fires per file, so one directive
anywhere in the project fails the whole run.

Which projects are affected:

```bash
grep -rl 'ditaa' xo-*/docs/*.rst | cut -d/ -f1 | sort -u
```

## Root cause

`sphinxcontrib/ditaa.py:113-120`:

```python
if format == 'html':      rel_imgpath = relative_uri(...)
elif format == 'latex':   rel_imgpath = relative_uri(...)
else:
    app.builder.warn('gnuplot: the builder format %s is not officially supported.' % format)
```

Two defects in that else branch, and a fix must address both:

1. `Builder.warn()` was deprecated in Sphinx 1.6 and removed in 2.0. Confirm it
   is gone:

   ```bash
   SP=$(dirname $(readlink -f $(command -v sphinx-build)))/../lib/python3.12/site-packages
   grep -n 'def warn' $SP/sphinx/builders/__init__.py   # no match
   ```

2. `rel_imgpath` is never assigned in that branch, so even with a working
   warning call the next use of it (`outrelfn = posixpath.join(rel_imgpath, ...)`,
   `ditaa.py:123`) raises `UnboundLocalError`.

The branch has therefore never worked for any builder. Supporting evidence that
it is unexercised copy-paste: the message says **gnuplot** — the extension
derives from `sphinxcontrib-gnuplot` and the string was never updated.

## Why CI never sees it

The xo docs build uses the html builder only:

```bash
grep -rn 'pseudoxml\|-b text\|-b man\|-b linkcheck' --include='*.cmake' --include='*.txt' xo-cmake/ | grep -v '/\.build'
```

(no matches; `xo_docdir_sphinx_config` in `xo-cmake/cmake/xo_macros/xo_cxx.cmake`
invokes `sphinx-build -b html`). `nix-build ci.nix -A xo-arena` is green.

## Who trips it

emacs flycheck, on every rst edit. Its `rst-sphinx` checker runs the pseudoxml
builder — `flycheck.el:11519-11529`:

```elisp
:command ("sphinx-build" "-b" "pseudoxml" "-q" "-N" ... source-original)
```

Each check leaves a crash dump:

```bash
ls -1 /tmp/sphinx-err-*.log | wc -l
```

## Corrected diagnosis — keep this, the wrong reading is the tempting one

First reading was "it fires on the six arena pages that carry a ditaa
directive". Plausible: the traceback names `render_ditaa_images`, and the crash
in the log happened while reading `AllocInfo-reference`, which does carry one.
It is wrong — the two commands above show a ditaa-free page in the same project
failing, and the trigger is project-wide, not per-file. The distinction matters
for the workaround: disabling the checker per-file would not have helped.

## Fix

```python
else:
    logger.warning('ditaa: builder format %s is not supported; skipping diagram.', format)
    rel_imgpath = ''
```

with `logger = logging.getLogger(__name__)` from `sphinx.util.logging`. Whether
a skipped diagram should also return early rather than write files nobody reads
is worth deciding while patching.

Routes, in preference order:

1. upstream PR to sphinx-contrib/ditaa, then drop the nixpkgs patch on the next
   release
2. nixpkgs patch now, upstream after — the package is ours to patch, but an
   overlay entry cuts against [[prefer-vanilla-nixpkgs]] if carried locally
   rather than landed in nixpkgs

Until then the workaround is `(setq-default flycheck-disabled-checkers '(rst-sphinx))`;
it costs nothing, since the checker crashes before emitting a single diagnostic.
