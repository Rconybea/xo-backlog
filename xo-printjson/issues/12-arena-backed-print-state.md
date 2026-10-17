# 12 -- arena-backed print state: frame stack and cycle-detection map

Status: open
Type: refactor
Raised: 2026-10-17 (RC), from `.xo-backlog/xo-printjson/issues/11`

Two heap containers hold the state of a json print.  Replace both with arena
containers from xo-arena (RC):

| state | today | replacement |
|---|---|---|
| `JsonPpSink`'s frame stack -- one entry per open object / list | `std::vector<Frame>` | `DArenaVector` |
| `JsonPrintState`'s cycle detection -- every object met in this print, by address | `std::unordered_map<void const *, ObjectEntry>` | `DArenaHashMap` |

```bash
cd "$(xo-sdlc --umbrella2)"
grep -n 'stack_;' xo-printjson/include/xo/printjson/JsonPpSink.hpp
grep -n 'objects_;' -B4 xo-printjson/include/xo/printjson/JsonPrintState.hpp
ls xo-arena/include/xo/arena/ | grep -E 'DArenaVector|DArenaHashMap'
grep -n 'xo_dependency(${SELF_LIB} xo_arena)' xo-printjson/src/printjson/CMakeLists.txt   # already a dependency
```

The map's own comment already names this as the intent ("a DArenaHashMap
from a pool of temporary arenas is the intent", from
`.xo-backlog/xo-printjson/issues/02`).

## Why

The fixed-reserve principle: state lives in arenas whose reserves are
stated up front and configured before `main()` hands off, so a print that
outgrows its expectations is noticed rather than silently growing the heap.
Both containers are per-print scratch -- created when a print starts,
dropped when it ends -- which is the arena's natural shape.

## Questions to settle when building

1. **Where the arenas come from.**  Per print (map an arena per
   `JsonPrintState` -- `DArena::map` costs a syscall), from a pool of
   temporary arenas (the issues/02 intent), or the per-thread `TempArena`
   (cheap; but a print allocates while a nested `tostr()` may too, and a
   container that grows must stay the arena's last allocation -- see
   `.xo-backlog/xo-arena/issues/06`).
2. **Reserve sizes, and who configures them.**  The stack is bounded by
   `PrintJson::max_depth` (default 1000) -- a reserve can be computed from
   it.  The map grows with the number of objects in one print (a websock
   server snapshot is the large case today); its reserve is a
   configuration choice, presumably on `PrintJsonConfig`.
3. **What exhaustion does.**  A print that outgrows a reserve: throw, or
   abort with a diagnosis like `max_depth` does today.

**Done when:**
- neither container uses the heap; both reserves configured up front
- output unchanged (all pins and goldens green)
- a test that exceeds each reserve shows the chosen failure
