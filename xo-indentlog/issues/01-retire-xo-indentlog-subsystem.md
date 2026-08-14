# 01 — retire the xo-indentlog subsystem from xo-umbrella2

Status: hypothesised
Type: refactor
Progress: grep -c 'add_subdirectory(xo-indentlog)' CMakeLists.txt

Since 2026-08-13 no subsystem depends on xo-indentlog:

```bash
xo-deps --users-of=xo-indentlog --format=names -q     # empty
```

The ten conversion tickets that got it there are listed in the Phase 3 section
of the `ppsink-levelization` notes; the last was
`.xo-backlog/xo-interpreter2/issues/02`. This ticket is the follow-on: take the
directory out of the umbrella.

**It is not a foregone conclusion.** xo-indentlog is the reference
implementation of the pretty-printer design that xo-ppsink / xo-indentlog2
descend from, it still builds, and it still has its own `utest/`. Removing it
costs nothing in dependencies and loses a working comparison point. Decide that
first; everything below assumes the answer is "remove".

## The git-subrepo question, answered

`xo-indentlog/` is a git-subrepo (63 of the umbrella's directories are):

```bash
cat xo-indentlog/.gitrepo
#   remote = git@github.com:Rconybea/xo-indentlog.git
#   branch = main   commit = c93f231d   parent = 5e18fcc8   method = merge
```

**Removal needs no special command.** git-subrepo keeps *all* its state in
`<subdir>/.gitrepo` — there is no `.gitmodules`, no `.git/config` section, no
index entry outside the directory. Confirmed: `ls .gitmodules` -> no such file,
and `git subrepo` offers **no `remove`/`rm` subcommand** at all:

```bash
git subrepo 2>&1 | sed -n '/Commands:/,/^$/p'
#   clone init pull push | fetch branch commit | status clean config | help version upgrade
```

So the removal is a plain `git rm -r xo-indentlog`. That is the whole answer to
"is there anything special" — **except for one thing, which is live right now.**

### THE precaution: six unpushed subrepo commits

`git subrepo status` looks clean, because `Pulled Commit` equals `Upstream Ref`:

```bash
git subrepo status xo-indentlog
#   Upstream Ref:    c93f231d
#   Pulled Commit:   c93f231d
```

but that only says nothing has been *pulled* since. It says nothing about
umbrella commits made to the subdirectory since the last **push**. There are six:

```bash
git log --oneline 5e18fcc83f03da4997ee0e93d4e5cf3d48117b66..HEAD -- xo-indentlog/
#   9b6ed131 xo-indentlog: clang-format include order [TIDY]
#   c7705f08 xo-indentlog: include order patch for clang [BUGFIX]
#   65f25254 xo-indentlog: clang-format includes [TIDY]
#   f69e5aaa build: rely on XO_DEPENDENCY_BLOCK for xo-facet + 3 others [BUILD]
#   58571749 xo-indentlog: bugfix: missing self-dependency [BUILD]
#   fcb28405 xo-indentlog relies on xo-timeutil + bugfix pkgs/xo-indentlog2.nix
#   85f521de git subrepo push xo-indentlog          <- the last push
```

(`5e18fcc8` is the `parent` field in `.gitrepo` — the umbrella commit the last
pull was based on. The push itself is the seventh entry, hence six real ones.)

Two of those are substantive, not formatting: `58571749` (missing
self-dependency) and `fcb28405` (indentlog now relies on xo-timeutil).

**So: `git subrepo push xo-indentlog` BEFORE `git rm -r`.** Otherwise the
standalone `Rconybea/xo-indentlog` repo never receives them. They would remain
recoverable from umbrella history, but nobody cloning the standalone repo would
ever see them, and after removal there is no subrepo left to push them with.

This is worth generalising: **`git subrepo status` showing pulled == upstream is
not evidence that local work has been pushed.** Check the log against
`.gitrepo`'s `parent` before removing any subrepo.

### What removal does and does not do

- **Upstream is untouched.** `Rconybea/xo-indentlog` keeps its own history and
  goes on existing. Archiving or deleting it on GitHub is a separate decision.
- **Umbrella history is untouched.** `git log -- xo-indentlog/` still works
  after the directory is gone; nothing is rewritten.
- **Reversible.** Re-adding is one command:
  `git subrepo clone git@github.com:Rconybea/xo-indentlog.git xo-indentlog`.
  That is the strongest argument for just doing it rather than agonising.

## Umbrella wiring to unpick

Every place the umbrella names xo-indentlog, outside the directory itself and
outside other subrepos' own vendored CI (see the caveat below):

| file | line | what |
|---|---|---|
| `CMakeLists.txt` | 85 | `add_subdirectory(xo-indentlog)` |
| `CMakeLists.txt` | 172 | `indentlog` in `xo_umbrella_doxygen_deps(...)` |
| `xo.nix` | 31 | `xo-indentlog = callPackage pkgs/xo-indentlog.nix {...}` |
| `ci.nix` | 21 | `xo-indentlog` in the `inherit (xoPkgs)` list |
| `pkgs/xo-indentlog.nix` | — | the package; deletes |
| `pkgs/xo-userenv.nix` | 26, 75 | argument + `paths` entry |
| `pkgs/xo-userenv-slow.nix` | 27, 46, 107 | argument + `xo_indentlog_src` + build line |
| `shells.nix` | 344 | `indentlog = pkgs.xo-indentlog;` |
| `shells.nix` | 19 | a `git subtree split --prefix=xo-indentlog` comment |
| `.forgejo/workflows/ci.yaml` | 71-75 | `nix-build ci.nix -A xo-indentlog` step |
| `.forgejo/workflows/ci-cmake.yaml` | 128-143 | configure/build/utest/install steps |
| `README.md` | 167 | `nix-build -A xo.indentlog` example |
| `xo-cmake/etc/xo/subsystem-edges` | — | **generated**; re-capture, do not edit |

The two `pkgs/xo-userenv*.nix` entries are the only **live** nix references left
(the commented-out ones in xo-callback / xo-printjson / xo-websock / xo-webutil /
xo-reactor were removed 2026-08-13). They are deliberate: those are aggregate
environments that *install* and *build* indentlog, not consumers of its headers.
That is why they survived the dependency migration and why they belong here.

**Caveat on grepping for this.** A naive tree-wide grep returns ~200 hits, and
the large majority are **other subrepos' own `.github/workflows/*.yml`** — each
satellite repo builds indentlog from GitHub when it is built standalone. Those
are the satellites' content, not umbrella wiring, and they are the satellites'
problem, not this ticket's. Likewise `xo-unit/{flake.nix,pkgs/*}` — xo-unit
vendors its own package set. Filter with `grep -v '/\.github/'` and
`grep -v '^xo-unit/'` before reading the list.

## Three source files still include it — resolve these first

```bash
grep -rl '#include <xo/indentlog/' --include=*.hpp --include=*.cpp xo-*/ \
  | grep -v '/\.build/' | grep -v '^xo-indentlog/'
#   xo-object2/utest/X1Collector.test.cpp
#   xo-object2/utest/Printable.test.cpp
#   xo-refcnt/include/xo/refcnt/Refcounted_indentlog.hpp
```

None is compiled today (the first two are commented out of
`xo-object2/utest/CMakeLists.txt:10-11` as "moved to xo-gc/"; the third is
included by nothing), so none blocks the build. But leaving a file that includes
a header from a deleted repo is a trap for the next reader, so each needs a
decision — see `.xo-backlog/xo-interpreter2/issues/02` for the detail, including
the open question of where `Printable.test.cpp`'s coverage went (there is **no**
`Printable.test.cpp` in xo-gc).

`Refcounted_indentlog.hpp` is the interesting one: it is a deliberate opt-in
adapter for a consumer that has legacy indentlog, and
`xo-refcnt/src/CMakeLists.txt:12-16` carries a comment explaining why xo-refcnt
deliberately does *not* declare the dependency. With xo-indentlog gone from the
umbrella that header serves no in-tree consumer — but xo-refcnt is itself a
subrepo that is built standalone, where the adapter may still make sense. Decide
in xo-refcnt's terms, not the umbrella's.

## Suggested order

1. **decide** whether to retire at all (see the top of this ticket)
2. `git subrepo push xo-indentlog` — the six commits above
3. resolve the three source files
4. unpick the wiring table, `pkgs/xo-indentlog.nix` last
5. `git rm -r xo-indentlog`
6. **re-capture** `subsystem-edges` (`.build/reconfigure`, then
   `--capture-subsystem-edges`, then reinstall xo-cmake)
7. `xo-build --sweep` — expect **61** subsystems attempted, not 62; and
   `nix-build ci.nix -A xo-userenv` (the aggregate is the thing most likely to
   still name it)

Step 7's count change is the check that the removal actually landed: the sweep
line has read `62 attempted` throughout this whole migration.
