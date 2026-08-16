# 03 — `--sweep` compiles two of xo-imgui's five examples

Status: fixed 2026-08-15
Type: bug

Found 2026-08-15 while converting xo-imgui for
`.xo-backlog/tostr-arena/issues/02` (class B). Two separate problems that
conspire: the files contain a lot of dead code, and the build that would have
caught it isn't running.

## 1. Each file is several copies of an example

```bash
for f in xo-imgui/example/ex3/imgui_ex3.cpp xo-imgui/example/ex4a/imgui_ex4a.cpp; do
  printf "%-24s lines=%-6s int-main=%s\n" "${f##*/}" "$(wc -l < $f)" "$(grep -c '^int main' $f)"
done
#   imgui_ex3.cpp   lines=4925  int-main=3
#   imgui_ex4a.cpp  lines=4307  int-main=3
```

The structure is a concatenation. In both files a second copy begins partway
down, still carrying the *original file's* banner comment:

```bash
grep -n '^int main\|^/\* xo-imgui/example' xo-imgui/example/ex3/imgui_ex3.cpp
#   :2     /* xo-imgui/example/ex1/imgui_ex3.cpp
#   :1745  int main(int, char **)
#   :2146  /* xo-imgui/example/ex1/imgui_ex2.cpp     <- second copy starts, of a DIFFERENT example
#   :3892  int main(int, char **)
#   :4912  int main() {
```

ex4a has the same shape at `:2142` / `:3888` / `:4294`. Note the banner at
`:2146` says `imgui_ex2.cpp` — so ex3 contains a copy of ex2, and so does ex4a.
All three banners in each file also claim to live in `example/ex1/`.

**Exactly one `main` survives preprocessing.** Measured with the real compile
flags, from a Vulkan-enabled build directory:

```bash
cd <build>/example/ex3
FLAGS=$(grep -m1 '^CXX_FLAGS'    CMakeFiles/imgui_ex3.dir/flags.make | sed 's/^CXX_FLAGS = //')
INCS=$( grep -m1 '^CXX_INCLUDES' CMakeFiles/imgui_ex3.dir/flags.make | sed 's/^CXX_INCLUDES = //')
g++ -E $FLAGS $INCS <src>/xo-imgui/example/ex3/imgui_ex3.cpp | grep -c '^int main'
#   1
```

So roughly half of each file is unreachable, kept alive only by `#if` regions
that never open.

## 2. The dead code holds references that do not name anything

Four sites across the two files:

```bash
grep -n 'tostr0(' xo-imgui/example/ex3/imgui_ex3.cpp xo-imgui/example/ex4a/imgui_ex4a.cpp
#   ex3 :1848  :3995      ex4a :1848  :3991
#   all four:  std::string font_path = xo::tostr0(fonts_path, "/truetype/DejaVuSans.ttf");
```

`tostr0` is declared in **`xo::pp`**
(`xo-ppsink/include/xo/ppsink/tostr0.hpp:25`), never in `xo`. `xo::tostr0` does
not exist and never did — the legacy spelling was `xo::tostr`, from xo-indentlog,
which no subsystem has reached since 2026-08-13. These compile only because they
are dead:

```bash
g++ -E $FLAGS $INCS <src>/xo-imgui/example/ex3/imgui_ex3.cpp | grep -c 'tostr0'
#   0
```

That makes them invisible to every migration sweep that works by grep, and they
will keep showing up in `tostr0` censuses until the dead code goes. They were
"converted" once during class B and reverted the same session, because renaming
a symbol that does not exist to a different symbol that does not exist is not
progress.

## 3. Why nobody noticed: the local build has Vulkan off

`ex3`, `ex4`, and `ex4a` are each gated on `XO_ENABLE_VULKAN`
(`example/ex3/CMakeLists.txt:4`, `ex4/CMakeLists.txt:2`, `ex4a/CMakeLists.txt:4`),
and the cached value is off:

```bash
grep -n 'XO_ENABLE_VULKAN' xo-imgui/.build/CMakeCache.txt
#   :354  XO_ENABLE_VULKAN:BOOL=OFF
```

So `xo-build --sweep --with-examples` compiles only `ex1` and `ex2` — three of
the five examples are never built, and the sweep still reports xo-imgui `ok`.

**Vulkan is available in this shell**, so this is a stale cache rather than a
missing dependency:

```bash
pkg-config --modversion vulkan     # 1.4.313
```

Verified by configuring a throwaway tree with it on — all four remaining example
binaries build:

```bash
cmake -S xo-imgui -B <scratch> -DCMAKE_MODULE_PATH=$HOME/local/share/cmake \
      -DCMAKE_PREFIX_PATH=$HOME/local -DXO_ENABLE_EXAMPLES=1 \
      -DXO_ENABLE_VULKAN=ON -DXO_ENABLE_OPENGL=ON
cmake --build <scratch> -j8        # ex1 ex2 ex3 ex4 ex4a all link
```

A throwaway directory rather than `xo-build --sweep -- -DXO_ENABLE_VULKAN=ON`
deliberately: `--` flags stick in the cache (`CONVENTIONS.md`), and the point
here was to measure, not to change the tree's configuration.

This is the third time in this repo that a build which *looks* green has been
covering less than assumed — see `01-example-guard-mismatch.md` here, and the
`--with-examples` history in `CONVENTIONS.md`. The pattern each time: a guard
whose value nobody re-checked.

## Files

- `xo-imgui/example/ex3/imgui_ex3.cpp` — 4925 lines, 3 `main`, dead copy from `:2146`
- `xo-imgui/example/ex4a/imgui_ex4a.cpp` — 4307 lines, 3 `main`, dead copy from `:2142`
- `xo-imgui/example/{ex3,ex4,ex4a}/CMakeLists.txt` — the `XO_ENABLE_VULKAN` gates
- `xo-imgui/.build/CMakeCache.txt:354` — the stale value

## Done when

- a default `xo-build --sweep --with-examples` compiles all five examples, or it
  is recorded here why three of them deliberately do not build by default
- `xo-build --sweep` still reads
  `62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`

The dead `#ifdef DEBUG` blocks are **not** part of this: see the decision at the
end of this ticket.

## Notes

**Deleting is the whole job, but check before you delete.** The dead halves are
copies of `imgui_ex2.cpp` rather than of their own host, so they are not a
backup of anything: `example/ex2/imgui_ex2.cpp` exists, is live, and is 1/4 the
size. Worth a `diff` against it before removing, in case the copy carries a
change that never made it back.

**`nix-build ci.nix -A xo-imgui` is unavailable as a check right now** — it
fails on a fixed-output hash mismatch fetching `amf-headers-1.4.36`, reached via
ffmpeg -> chromaprint -> pipewire -> sdl3 -> sdl2-compat. That is an upstream
nixpkgs fetch, unrelated to xo, but it means the umbrella build is the only
coverage these examples have, which is exactly what makes the Vulkan gate above
worth fixing first.

## CORRECTED 2026-08-15, same day — both mechanisms were mis-stated

### The dead region is one `#ifdef DEBUG`, not scattered `#if` blocks

Line 1 of each file opens it, and it closes ~4290 lines later:

```bash
sed -n '1p' xo-imgui/example/ex4a/imgui_ex4a.cpp        # #ifdef DEBUG
sed -n '4291p' xo-imgui/example/ex4a/imgui_ex4a.cpp     # #endif //DEBUG
sed -n '4293p' xo-imgui/example/ex3/imgui_ex3.cpp       # #endif
grep -o '\-DDEBUG[^ ]*' xo-imgui/.build/example/ex3/CMakeFiles/*/flags.make
#   (nothing -- DEBUG is never defined by the build)
```

So it is not two half-live copies: **everything above the `#endif` is dead, and
the live program is what follows it.** Confirmed by which source lines survive
preprocessing:

```bash
g++ -E $FLAGS $INCS .../imgui_ex3.cpp | grep -o '^# [0-9]* "[^"]*imgui_ex3.cpp"' \
  | awk '{print $2}' | sort -n | head -1     # first surviving line: 4296
```

| file | lines | dead | live |
|---|---|---|---|
| `ex3/imgui_ex3.cpp` | 4925 | 1-4293 (87%) | 4295-4925, `main` at `:4912` |
| `ex4a/imgui_ex4a.cpp` | 4307 | 1-4291 (99.6%) | 4293-4307, `main` at `:4294`, just `MinimalImGuiVulkan app` over `VulkanApp.hpp` |

That makes the fix smaller and safer than this ticket first suggested: delete
from line 1 to the `#endif`. The two `int main` definitions and both
`xo::tostr0` sites in each file are inside that span, so they go with it, and
the `Progress:` count falls to 0 in one edit per file.

The **"is it a copy of ex2"** question still stands — the banner at `ex3:2146`
says `imgui_ex2.cpp` — but it is now a question about the contents of a block
that is provably unreachable, not about half the program.

### "Stale cache" was wrong: Vulkan is off by default in every per-subsystem build

This ticket said `xo-imgui/.build/CMakeCache.txt:354` held a stale value. It
does not. Clobbering and recreating the directory produces `OFF` again, because
that is the default and `xo-build` only overrides it on request:

```bash
grep -n 'XO_ENABLE_VULKAN' xo-cmake/cmake/xo_macros/xo_cxx.cmake
#   :3  option(XO_ENABLE_VULKAN "enable vulkan dependency for imgui apps" OFF)
xo-build --help | grep -n 'with-vulkan'
#   --with-vulkan=VKFLAG    in configure step, set -DXO_ENABLE_VULKAN=VKFLAG [OFF]
```

The **umbrella** `.build` has it on (`XO_ENABLE_VULKAN:BOOL=1`), which is why the
two disagree and why the setting looked deliberate-then-lost. It never was: any
`xo-build` invocation without `--with-vulkan=ON` — including
`xo-build --sweep` — configures xo-imgui with Vulkan off and compiles two of the
five examples.

```bash
xo-build -q --configure --with-utests --with-examples --with-vulkan=ON --build --install xo-imgui
#   all five of ex1 ex2 ex3 ex4 ex4a link
```

**Why the wrong reading was plausible:** the umbrella cache genuinely has it on,
and a note in this repo's working memory says Vulkan is deliberately enabled —
both true, and both about the umbrella build rather than the per-subsystem one
that `--sweep` actually uses. Checking `CMakeCache.txt` answered "what is the
value" and was mistaken for an answer to "where does the value come from".

So the fix is not to edit a cache. Either `--sweep` should pass
`--with-vulkan=ON`, or the examples' guard should not be the thing that decides
whether they compile — which is the question already open in
`02-build-without-vulkan.md`.

### ex4 is fine

Checked at the same time: `example/ex4/` is a proper multi-file example — a
323-line `imgui_ex4.cpp` plus `DrawState.cpp` (1119), `AppState.cpp`,
`GenerationLayout.cpp`, `GcStateDescription.cpp`, `AnimateGcCopyCb.cpp` and
their headers. One `main`, no dead block, no `tostr0`. Its class-B conversion
compiles under `--with-vulkan=ON`. Nothing to do here.

## DECIDED 2026-08-15 by RC — the dead blocks stay, and their `tostr0`s are harmless

The `#ifdef DEBUG` regions in `ex3/imgui_ex3.cpp` and `ex4a/imgui_ex4a.cpp` are
**deliberate**: they are kept as context for the live example below them. They
are not to be deleted, trimmed, or diffed against `ex2` with a view to removing
them. Sections 1 and 2 above stand as a description of what is in those files;
they no longer describe anything to fix.

That leaves this ticket about one thing only: `--sweep` builds ex1 and ex2 and
silently skips ex3, ex4 and ex4a (section 3, as corrected).

### The `tostr0` calls in them were converted anyway (RC, same day)

The four sites at `ex3:1848,3995` and `ex4a:1848,3991` now read
`xo::pp::tostr(...)`. RC's reasoning: it does not matter that this code would
not compile if the guards were removed, because it never will be — and leaving a
`tostr0` spelling there means every future census has to know about the
exception.

Spelled `xo::pp::tostr` rather than `xo::tostr`, because the block is kept **as
context**: the point of reading it is to see what the example looked like, and
`xo::tostr` was the xo-indentlog spelling that no longer names anything either.
Only the calls changed — no include was added, since one would land in dead code.

Confirmed still dead, so nothing about the build changed:

```bash
g++ -E $FLAGS $INCS xo-imgui/example/ex3/imgui_ex3.cpp | grep -c 'tostr'   # 0
xo-build -q --configure --with-utests --with-examples --with-vulkan=ON --build --install xo-imgui
#   ok
```

So no census exception is needed after all. The `grep -vE 'ex3/imgui_ex3\.cpp|
ex4a/imgui_ex4a\.cpp'` filter in `.xo-backlog/tostr-arena/issues/02`'s
`Progress:` line is now redundant; harmless to leave, but it no longer excludes
anything.

### Why this was worth writing down rather than just leaving

A dead block containing a call to a nonexistent symbol looks exactly like an
unfinished migration, and this session converted it once before catching that.
Without this note the next sweep does the same thing.

## Fixed 2026-08-15 — `--sweep` now asks for vulkan when the host has it

`xo-build --sweep --with-vulkan=ON` already worked: the replay loop at
`xo-cmake/bin/xo-build.in:468-474` forwards every argument except `--sweep` and
`--clobber`. The gap was that nobody passed it, and nothing said so.

The sweep recipe now supplies it itself:

```bash
xo-build -n --sweep | head -3
#   xo-build: --sweep: vulkan available, adding --with-vulkan=ON (pass --with-vulkan=OFF to override)
#   ... -DXO_ENABLE_EXAMPLES=1 -DXO_ENABLE_VULKAN=ON
```

**Detected by capability, not by hostname.** `xo-imgui/src/imgui/CMakeLists.txt:38`
is `find_package(Vulkan REQUIRED)`, so forcing it on where vulkan is absent
turns a green sweep red — which on roly-laptop-26 would be a worse failure than
the one this ticket started with. The probe is `$VULKAN_SDK` or
`pkg-config --exists vulkan`; when neither holds, the sweep says so rather than
skipping three examples in silence:

```
xo-build: --sweep: no vulkan found; xo-imgui examples ex3/ex4/ex4a will not be compiled
```

An explicit `--with-vulkan=VKFLAG` always wins, in both directions — verified
that `--with-vulkan=OFF` still yields `-DXO_ENABLE_VULKAN=OFF` and suppresses
the message, and that `--with-vulkan=ON` does not produce the flag twice.

### Verified

```bash
xo-build -q --configure --build --install xo-cmake   # xo-build runs from the installed copy
xo-build -q --sweep
#   62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped
#   --sweep ok (build and utest)

ls xo-imgui/.build/example/*/imgui_*
#   ex1 ex2 ex3 ex4 ex4a -- all five, from a bare --sweep
grep XO_ENABLE_VULKAN xo-imgui/.build/CMakeCache.txt
#   XO_ENABLE_VULKAN:BOOL=ON
```

Note the whole tree is now configured with `-DXO_ENABLE_VULKAN=ON`, not just
xo-imgui — the flag goes to every subsystem's configure line. Harmless (nothing
else reads it) but worth knowing before wondering why an unrelated cache changed.

### Files

- `xo-cmake/bin/xo-build.in` — the detection block in the `--sweep` recipe, and
  the `--sweep` entry in `--help`

### What this does not fix

`XO_ENABLE_VULKAN` still defaults OFF for a plain
`xo-build --configure <sub>`, so a one-off build of xo-imgui still skips three
examples unless asked. Whether that default is right at all is
`02-build-without-vulkan.md`'s question, not this one's.
