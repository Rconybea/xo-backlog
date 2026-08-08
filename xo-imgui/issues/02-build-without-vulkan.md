# 02 — xo_imgui should be buildable without Vulkan

Status: open (decision needed; homework partly done)
Type: task

Today the whole `xo_imgui` library is gated on `XO_ENABLE_VULKAN`
(`src/imgui/CMakeLists.txt:17`). With Vulkan off, **no library is built at all**
— not even the parts that have nothing to do with Vulkan.

History: the library was originally written against OpenGL, then refactored to
take advantage of Vulkan. The gate reflects where it ended up, not necessarily
where it has to be. It may turn out that a useful non-Vulkan build isn't worth
having, in which case today's state is optimal — but that should be a decision,
not an accident.

Prompted by `01-example-guard-mismatch.md`: fixing that exposed that "Vulkan
off" means "xo-imgui contributes nothing", which nobody had had to think about.

## Homework already done (2026-08-08)

**The entanglement is much smaller than expected.** Vulkan references per file:

| file | vulkan refs | lines |
|---|---|---|
| `include/xo/imgui/ImRect.hpp` | **0** | 148 |
| `include/xo/imgui/ImScale.hpp` | **0** | 46 |
| `include/xo/imgui/ImSpan.hpp` | **0** | 24 |
| `src/imgui/ImRect.cpp` | **0** | 56 |
| `include/xo/imgui/VulkanApp.hpp` | 35 | 242 |
| `src/imgui/VulkanApp.cpp` | 103 | 760 |

The split is clean at file granularity: ~274 lines Vulkan-free, ~1000 lines
entirely Vulkan. There is no interleaving to untangle.

**The Vulkan-free part has no graphics-API dependency at all.** Its complete
include set:

```
ImSpan.hpp  <- imgui.h
ImScale.hpp <- ImSpan.hpp, imgui.h
ImRect.hpp  <- ImSpan.hpp, imgui.h, <algorithm>, <utility>
ImRect.cpp  <- ImRect.hpp
```

No Vulkan, no SDL2, no OpenGL, no GLEW. These are pure imgui geometry helpers —
they need only upstream `imgui.h`. So a non-Vulkan `xo_imgui` is not a porting
job; it is a CMake split.

**No OpenGL-era application class survives in history.** `git log -S 'OpenGLApp'`
over `xo-imgui/` returns nothing, so `VulkanApp` does not appear to have a
displaced OpenGL sibling to restore. Whatever the OpenGL original looked like,
what remains today is Vulkan-only. **Worth confirming** — the search was one
identifier against one path, and the class may have had another name.

**Related duplication, probably the same decision.** The imgui core sources
(`imgui.cpp`, `imgui_demo.cpp`, `imgui_draw.cpp`, `imgui_widgets.cpp`,
`imgui_tables.cpp`) are compiled **three times**: once into `libxo_imgui`, and
again into each of `imgui_ex1` and `imgui_ex2` (5 source entries in each of the
three CMakeLists). `example/ex1/CMakeLists.txt:6` already carries the note
`# TODO: maybe incorporate imgui sources into xo-imgui library`. A Vulkan-free
`xo_imgui` that carries imgui core would let both OpenGL examples drop their
duplicate source lists — which is likely the real payoff here, more than the
geometry helpers themselves.

## The decision

Sketch, not a recommendation — the point of the ticket is to choose deliberately:

1. **Split the target.** `xo_imgui` (imgui core + ImRect/ImScale/ImSpan,
   ungated) and `xo_imgui_vulkan` (VulkanApp, gated). Examples depend on
   whichever they need; ex1/ex2 stop rebuilding imgui core. Most work, best end
   state.
2. **Keep one target, narrow the gate.** Build `xo_imgui` always; add
   `VulkanApp.cpp` and `Vulkan::Vulkan` to it only under `XO_ENABLE_VULKAN`. A
   target whose contents vary by flag — cheap, but consumers can't tell what
   they're getting.
3. **Leave as is.** Accept that xo-imgui is a Vulkan subsystem. Then ex1/ex2
   should arguably move out of xo-imgui, since they are OpenGL programs living
   in a Vulkan subsystem and depending on none of it.

Option 3 is a legitimate answer and should not be dismissed — but note it argues
for *moving the examples*, which is itself work.

## Open questions for the decision

- Is there a real consumer for a non-Vulkan `xo_imgui`, or are ex1/ex2 the only
  ones? Nothing outside `xo-imgui/` depends on it today.
- Should imgui core live in the library (killing the 3× duplication)? If yes,
  that mostly settles the question in favour of option 1 or 2.
- Does anything besides `VulkanApp` need Vulkan? Measured: no.
- Is `XO_ENABLE_VULKAN` meant to mean "Vulkan is available" or "build the Vulkan
  app"? They are currently conflated.

**Files:**
- `xo-imgui/src/imgui/CMakeLists.txt:17` — the gate
- `xo-imgui/include/xo/imgui/{ImRect,ImScale,ImSpan}.hpp`, `src/imgui/ImRect.cpp`
  — the Vulkan-free part
- `xo-imgui/include/xo/imgui/VulkanApp.hpp`, `src/imgui/VulkanApp.cpp` — the
  Vulkan part
- `xo-imgui/example/{ex1,ex2}/CMakeLists.txt` — duplicate imgui core sources

**Done when:**
- a decision is recorded here with its reasoning, and either implemented or
  explicitly declined
- if implemented: `XO_ENABLE_VULKAN=OFF` yields a usable `xo_imgui`, and both
  guard settings build clean with `--with-examples`

## Notes

Verify any change in **both** guard settings. `01` was a bug that existed in only
one of four `XO_ENABLE_{EXAMPLES,OPENGL,VULKAN}` combinations, and the default
local build was not that combination.
