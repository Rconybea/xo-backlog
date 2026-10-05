# 04 -- reflect std::atomic, std::unique_ptr, enums, std::deque

Status: open
Type: feature
Milestone: reflection-driven-json

These have no `EstablishTdx` specialisation (`xo-reflect/include/xo/reflect/
Reflect.hpp`), so they reflect as opaque `AtomicTdx` leaves. The websock
printers therefore summarise them by hand at each use (`member_as`):
- `open_` and `listen_port_`, atomics, as `.load()`;
- `kind_` and `state_`, enums, via descr functions;
- `readjson_`, a `unique_ptr`, as "set" / "null";
- `outbound_q_`, a deque, as "N queued".

Proposed:
- `std::atomic<T>` reflects as its `load()`ed `T`;
- `std::unique_ptr<T>` as `mt_pointer`, like `rp` / raw;
- `std::deque<T>` as `mt_vector`;
- enums by name, given a name table, or as their integer.

Each changes how existing generic output renders these types: check the
consumers per type.

Out of scope: maps (`issues/05`); `std::function` and `std::regex`, which
have nothing structural to show (keep a summary printer, or reflect them
as presence / capture count, to decide).

**Done when:** websock needs no `member_as` for these four, and each has a
reflect test.
