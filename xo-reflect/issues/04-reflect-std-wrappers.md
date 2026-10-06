# 04 -- reflect std::atomic, std::unique_ptr, std::deque

Status: open (unique_ptr done 2026-10-06 -- umbrella `1812a593`, `f98dfd12`; atomic, deque to come)
Type: feature
Milestone: reflection-driven-json

These have no `EstablishTdx` specialisation (`xo-reflect/include/xo/reflect/
Reflect.hpp`), so they reflect as opaque `AtomicTdx` leaves. The websock
printers therefore summarise them by hand at each use (`member_as`):
- `open_` and `listen_port_`, atomics, as `.load()`;
- `kind_` and `state_`, enums, via descr functions (now `issues/06`);
- `readjson_`, a `unique_ptr`, as "set" / "null";
- `outbound_q_`, a deque, as "N queued".

Proposed:
- `std::atomic<T>` reflects as its `load()`ed `T`;
- `std::unique_ptr<T>` as `mt_pointer`, like `rp` / raw;
- `std::deque<T>` as `mt_vector`.

Each changes how existing generic output renders these types: check the
consumers per type.

Out of scope: enums, split out to `issues/06` (RC, 2026-10-05); maps
(`issues/05`); `std::function` and `std::regex`, which
have nothing structural to show (keep a summary printer, or reflect them
as presence / capture count, to decide).

**Done when:** websock needs no `member_as` for these three (atomics,
`unique_ptr`, deque), and each has a reflect test.

## std::unique_ptr done, 2026-10-06 -- umbrella `1812a593`, `f98dfd12`

**xo-reflect (`1812a593`).**
- `EstablishTdx<std::unique_ptr<T, D>>` requires the cv-stripped `T` and
  returns `RefPointerTdx<std::unique_ptr<T, D>>`. No new Tdx class:
  `RefPointerTdx` needs only `element_type`, `get()` and a test for null,
  and its comment now says it serves any smart pointer.
- It strips cv from the pointee, as `RawPointerTdx` does (`pointee_t`,
  plus a `const_cast` on read), so `unique_ptr<const T>` shares `T`'s
  descriptor. Every existing use is a non-const `rp<T>`, so none changes.
- `std::unique_ptr<T[], D>` stays atomic, through a more specialised
  `EstablishTdx`: it owns an array of unknown length, not one pointee.
- The deleter is not reflected.
- Tests (`PointerTdx.test.cpp`, `[uniqueptr]`):
  - null has 0 children; non-null has 1, the pointee;
  - `unique_ptr<const T>`;
  - a custom deleter;
  - `unique_ptr<T[]>` is atomic.

  printjson (`PrintJson.test.cpp`): a `unique_ptr` member prints its pointee
  in full, or `null`.
- No reflected `unique_ptr` member existed anywhere (checked: LocalEnv,
  VsmStackFrame, BrownianMotion, websock, printjson). One side effect: the
  declared type of `readjson_`'s `member_as`,
  `std::unique_ptr<Json::CharReader>`, now has metatype `pointer` in the
  golden snapshot.

**websock (`f98dfd12`), RC's option (b).**
- `Json::CharReader`, a third-party type, is reflected with no members, in
  `WsSessionRouter::reflect_self`. `WsSessionRouter::readjson_` is a
  reflected member, and its `member_as` (`"set"` / `"null"`) is gone.
- It prints as `{"_name_": "CharReader", .., "_members_": []}`, or `null`.
- Golden diff, checked by script: apart from `readjson_`, the snapshots
  are identical up to a consistent renumbering of ids (the two new
  CharReader objects take ids).
- introspect nests only a struct that has members, so the row reads
  `readjson_: CharReader`.
- Tests updated: the router checks in `WebserverLive.test.cpp`, now by
  name; `router_expand.mjs`.

Checked: ctest 49 / 49; the 20 browser tests; `xo-build --sweep`.

**Left in this ticket:** `std::atomic<T>` (`open_`, `listen_port_`) and
`std::deque<T>` (`outbound_q_`, which is also lock-guarded:
`xo-websock/issues/15`).
