# 02 — `reconfigure --capture-subsystem-edges`, with a provisioning check

Status: open
Type: feature

Refreshing `xo-cmake/etc/xo/subsystem-edges` is currently a three-step manual
recipe, and its one precondition is invisible:

```bash
cmake -B .build -S .
cp .build/xo-subsystem-edges.txt xo-cmake/etc/xo/subsystem-edges
xo-build --configure --build --install xo-cmake    # xo-build reads the INSTALLED copy
```

Nothing checks that the build directory was fully provisioned, and nothing
reminds you about the third step. Both failure modes are silent.

## Why a flag rather than a documented recipe

**A `xo_dependency()` call inside a config-time guard emits no edge when that
guard is off**, so copying from a partial build silently *truncates* the graph.
Measured 2026-08-08:

```bash
# count xo_dependency-family calls by enclosing config-time guard
grep -rn "xo_dependency\|xo_headeronly_dependency\|xo_pybind11_dependency" \
     --include=CMakeLists.txt . | grep -v '\.build'
```

| guard | calls under it |
|---|---|
| `XO_ENABLE_EXAMPLES` | 88 |
| `ENABLE_TESTING` | 45 |
| `XO_ENABLE_VULKAN` | 6 |
| (unguarded) | 195 |

So **139 of 334** dependency declarations — 42% — are behind a switch.
`XO_ENABLE_DOCS`, `XO_ENABLE_ASM` and `XO_ENABLE_OPENGL` currently gate none,
though OPENGL nests inside EXAMPLES and structurally could.

A truncated graph is worse than a stale one: `xo-build --with-deps` would build
too little, and the failure appears far away as a missing header or an
unresolved symbol in a subsystem nobody edited.

## Why `reconfigure` is the right home

`.build/reconfigure` is already generated per build directory from
`xo-cmake/share/xo-macros/xo-reconfigure.in` (via
`xo_generate_reconfigure_script`, `xo_cxx.cmake:298`), and `configure_file`
already substitutes **exactly the values the check needs**:

```
@CMAKE_SOURCE_DIR@   @CMAKE_BINARY_DIR@
@ENABLE_TESTING@     @XO_ENABLE_EXAMPLES@   @XO_ENABLE_VULKAN@
@XO_ENABLE_OPENGL@   @XO_ENABLE_DOCS@       @XO_ENABLE_ASM@
```

So the script knows, without asking cmake again, both where to copy from/to and
whether this build directory is entitled to do so. No new plumbing.

It also already has the argument-parsing shape to extend (`--print-source-dir`,
`--print-build-dir`, `--clobber`, `--enable-testing`).

## Sketch

```
reconfigure --capture-subsystem-edges [--force]
```

1. Verify `@CMAKE_BINARY_DIR@/xo-subsystem-edges.txt` exists and is newer than
   the last configure; if not, say so and exit non-zero.
2. **Check provisioning.** With any of `ENABLE_TESTING`, `XO_ENABLE_EXAMPLES`,
   `XO_ENABLE_VULKAN` off, refuse and name which:
   ```
   reconfigure: refusing to capture subsystem edges from an under-provisioned build
     ENABLE_TESTING=1  XO_ENABLE_EXAMPLES=0  XO_ENABLE_VULKAN=0
   88 + 6 dependency declarations are behind the disabled switches and would be
   omitted from the captured graph.
   Reconfigure with them enabled, or pass --force if you know better.
   ```
3. Copy to `@CMAKE_SOURCE_DIR@/xo-cmake/etc/xo/subsystem-edges`.
4. **Diff and report** what changed — the useful review artifact, and how the
   two missed edges below would have been noticed.
5. Remind that `xo-build` reads the *installed* copy, so `xo-cmake` must be
   reinstalled for the change to take effect. (Or do it — see questions.)

## Motivating incident

On 2026-08-08 the edges were hand-edited across the xo-simulator, xo-process,
xo-tokenizer, xo-pyreactor, xo-expression/xo-reader and xo-interpreter
migrations. The hand passes:

- **missed two edges** — `xo-indentlog2 xo-expression` and
  `xo-indentlog2 xo-reader`, both from *test-only and example-only* deps,
  because the editor was thinking about library deps
- **mis-sorted new entries twice**, needing two follow-up corrections

Copying the generated file fixed both classes at once. The generated file has no
blind spot: it records whatever `xo_dependency()` actually ran.

## Questions to settle

- **Should it also reinstall xo-cmake?** Convenient, but a capture flag that
  silently triggers a build is surprising. A printed reminder may be better —
  or a separate `--install-after`.
- **What counts as "adequately provisioned"?** The three measured switches are
  today's answer, but the check should ideally be *derived* rather than
  hardcoded — e.g. cmake records which guards it saw dependency calls under.
  Hardcoding three names means the check silently stops being complete when a
  fourth guard appears. Worth thinking about before implementing.
- **Should `--force` exist at all?** It permits exactly the corruption the
  feature prevents. If it does, the captured file should carry a comment saying
  it was force-captured and with which switches off.
- **Does the standalone (non-umbrella) case make sense?** A satellite build
  knows only its own edges, so capture should probably refuse outside the
  umbrella toplevel — `xo_generate_reconfigure_script` already distinguishes
  those cases.

**Files:**
- `xo-cmake/share/xo-macros/xo-reconfigure.in` — the template; argument parsing
  and the `@...@` substitutions are both already there
- `xo-cmake/cmake/xo_macros/xo_cxx.cmake:298` — `xo_generate_reconfigure_script`
- `xo-cmake/etc/xo/subsystem-edges` — the destination
- `xo-cmake/bin/xo-build.in:266` — reads the *installed* copy, hence step 5

**Done when:**
- refreshing the graph is one command from a fully-provisioned build
- attempting it from an under-provisioned one fails loudly and names the
  missing switches
- the recipe in `.xo-backlog/github-workflows/issues/01-align-with-forgejo.md`
  is replaced by that command

## Notes

Do **not** wire this into CI. CI build directories are not reliably
fully-provisioned, and an automatically-captured truncated graph would be
committed without anyone reviewing the diff. The value here is a local
operation that refuses to do the wrong thing — see
`.xo-backlog/CONVENTIONS.md` on tools that silently no-op or silently truncate.
