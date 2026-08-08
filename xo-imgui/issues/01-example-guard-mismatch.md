# 01 — imgui examples linked a library they never used

Status: fixed 2026-08-08
Type: bug

With `--with-examples` and OpenGL enabled but Vulkan disabled, `xo-imgui` failed:

```
ld: cannot find -lxo_imgui: No such file or directory
gmake[2]: *** [example/ex1/CMakeFiles/imgui_ex1.dir/build.make:215: example/ex1/imgui_ex1] Error 1
```

Same for `imgui_ex2`.

## Cause — a spurious dependency, not a guard mismatch

**This ticket originally blamed a guard mismatch. That diagnosis was wrong**, and
the fixes it proposed (gate the examples on `XO_ENABLE_VULKAN`, or widen the
library's guard) would each have been wrong too — the first would have disabled
two working OpenGL examples. Recorded here because the wrong reading was
plausible and someone may reach for it again.

The guards are *correct* and deliberately independent:

| | guard | what it is |
|---|---|---|
| library `xo_imgui` (`src/imgui/CMakeLists.txt:17`) | `XO_ENABLE_VULKAN` | `VulkanApp.cpp` + `imgui_impl_vulkan.cpp`, links `Vulkan::Vulkan` — genuinely Vulkan-only |
| ex1, ex2 | `XO_ENABLE_OPENGL` | compile their own imgui core + `imgui_impl_sdl2/opengl3`, link `OpenGL::GL`, GLEW, SDL2 |

ex1/ex2 are OpenGL examples that *should* build with Vulkan off. They failed
only because each carried

```cmake
xo_self_dependency(${SELF_EXE} xo_imgui)
```

while using **nothing** from that library. With Vulkan off the target does not
exist, so CMake treats the name as a plain library and emits `-lxo_imgui` — the
"undefined target degrades to a raw `-lfoo`" failure also recorded in
`.xo-backlog/generated-find-dependency/issues/02-handwritten-config-drift.md`.

## Measured: only one example actually used the library

`#include` of any `xo/imgui/` header, per example:

| example | guard | had the dep | uses `xo/imgui/` |
|---|---|---|---|
| ex1 | OPENGL | yes | **no** |
| ex2 | OPENGL | yes | **no** |
| ex3 | VULKAN | yes | **no** |
| ex4 | VULKAN | yes | **yes** — `VulkanApp.hpp`, `ImScale.hpp`, `ImRect.hpp` |
| ex4a | VULKAN | yes | **no** — ships its *own* local `VulkanApp.cpp/.hpp` |

Four of five were spurious. Only ex1/ex2 *broke*, because they are the only ones
whose guard lets them build while the library's guard does not. ex3/ex4a carried
the identical latent bug, invisible only because Vulkan-gated examples and the
Vulkan-gated library happen to switch together.

**Fix:** removed the dependency from ex1, ex2, ex3, ex4a. ex4 keeps it.

Verified both directions:
- `XO_ENABLE_VULKAN=OFF`: ex1+ex2 build and link; ex3/ex4/ex4a and the library
  correctly skipped.
- `XO_ENABLE_VULKAN=ON`: all five examples and `libxo_imgui.so` build and link —
  proving the four removed deps contributed no symbols.

## Why it went unnoticed

`xo-build --all` and every per-subsystem verification loop used `--with-utests`
but **not** `--with-examples`, so no example in the tree had ever been compiled
locally. `nix-build` does build examples, which is how it surfaced — on
`xo-tokenizer`, whose `example/tokenrepl` still used legacy `xo::log_config`
after the rest of that subsystem had migrated. That was a real migration miss;
this imgui failure is older and independent, and was simply sitting behind the
same blind spot.

**Add `--with-examples` to umbrella verification runs.** As of 2026-08-08 the
whole tree builds with it enabled.

**Files:**
- `xo-imgui/example/{ex1,ex2,ex3,ex4a}/CMakeLists.txt` — dependency removed
- `xo-imgui/example/ex4/CMakeLists.txt` — dependency kept, genuinely used

## Notes

A dependency that is never used is not inert: it is a latent link failure that
fires the moment the target stops existing in some configuration. `xo_imgui` is
conditionally defined, which turned an unused edge into a hard build break in
exactly one of four guard combinations. Prefer checking `#include`s against
declared deps when a subsystem has any conditionally-created target.
