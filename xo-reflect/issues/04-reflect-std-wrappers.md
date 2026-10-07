# 04 -- reflect std::atomic, std::unique_ptr, std::deque

Status: open (unique_ptr, atomic done -- umbrella `1812a593`, `f98dfd12`, `db432d09`, `02181655`, `d78751f4`; deque to come, with `xo-websock/issues/15`)
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

## Related: receivers reflected, 2026-10-06 -- umbrella `660410a3`, `db4b7a1f`, `118998b0`

Prompted by RC's questions about `IntrospectReceiver` and
`member_ref<rp<StreamReceiver>>`. Both had been left out for reasons that
predate print-once (`xo-printjson/issues/02`).

- **`660410a3`, `db4b7a1f`:**
  - The endpoint printer's inline receiver object now writes `_members_`
    from `reflected_members(self, "_")`, through `self_tp()`, so an
    application's receiver shows its own members.
  - `IntrospectReceiver` reflects `websrv_`. It prints as
    `{"_ref_": <server's _id_>}`: a snapshot prints the server first. It
    had been left out because, before print-once, printing it in full
    nested the whole server inside its own endpoint.
  - `receiver.mjs` pins the ref and the row `websrv_: (→)`.
- **`118998b0`:**
  - **xo-webutil:** `StreamReceiver::reflect_self` reflects the interface
    with no members. An `rp<Base>` reaches its most-derived type only if
    `Base` is reflected as a self-tagging struct. xo-webutil has no setup
    hook of its own, so `websock_reflect_types` calls it.
  - **websock:** `DynamicEndpoint::receiver_` is reflected, and its
    `member_ref` is gone. It still prints as a ref to the inline receiver
    object. Golden: a reorder only.
  - **printjson: "a ref is always printable".** The test's `BoxReceiver`
    is unreflected (an atomic), but the endpoint writes it as an object
    through `open_object_at`. Two places did not account for that:
    - `print_node` checked for a printed entry only for types it treats as
      objects, so it wrote `<error-json-printer-not-found>` (invalid json).
      Now a value whose address and type match a printed entry is a ref,
      whatever its own type. That is sound because entries come only from
      `open_object` / `open_object_at`.
    - `JsonMembers` judged printability by the declared target type. Now it
      uses the new `JsonPrintState::is_printed(TaggedPtr)`, and `member()` /
      `member_as()` decide by value, as `reflected_members` already did.
      Side effect: a null pointer or an empty vector of an unprintable
      type prints as `null` / `[]`, not as an error entry.

    Pinned by `PrintJsonCycle.test.cpp`, through both `state.print` and
    `JsonMembers`.

Checked: ctest 49 / 49; the 20 browser tests; `xo-build --sweep`.

## std::atomic done, 2026-10-06 -- umbrella `db432d09`, `02181655`, `d78751f4`

RC chose design (A), a reflection capability rather than per-`T`
printjson printers, and named it `std_atomic_info()`.

**xo-reflect (`db432d09`).**
- `StdAtomicTdx` (`atomic/StdAtomicTdx.hpp/.cpp`): `mt_atomic`, with no
  children. A `std::atomic<T>` cannot be traversed in place: only `load()`
  reads it. It provides:
  - `value_td()`, the description of T;
  - `value_size()`, `value_align()`;
  - `load(atomic, dst)`, which copies the current value into a buffer.
    std::atomic requires T to be trivially copyable, so the copy is
    sound.
- `EstablishTdx<std::atomic<T>>` installs it, with a `static_assert` that
  T is trivially copyable.
- `TypeDescrExtra::std_atomic_info()` (`nullptr` by default) and
  `is_std_atomic()`, forwarded by `TypeDescr`.
- Tests (`StdAtomic.test.cpp`): bool, int64, double, a later store, an
  atomic of a reflected enum; a plain int is not a std::atomic.

**printjson (`02181655`).** With no printer, a `std::atomic<T>` is
loaded into a buffer (64 bytes on the stack, else an aligned allocation)
and printed by `print_value` as its T, with no identity: it is a copy.
`JsonMembers::printable` accepts an atomic of a printable T. An atomic
pointer is deliberately not printable: its type cannot say whether its
target prints. Test: atomic int, bool and enum members print `7`, `true`
and `"calm"`. Not covered: the heap path (T over 64 bytes), which needs
libatomic.

**websock (`d78751f4`).** `WsSessionSender::open_` and
`WebserverImpl::listen_port_` are reflected, and their `member_as` are
gone. The server printer's `_members_` is now just
`reflected_members(tp, "_")`. Golden diff, checked by script: identical up
to member order; in each sender, `open_` moves ahead of `target_`. Tests:
the sender checks in `WebserverLive.test.cpp`, and `sender_expand.mjs`.

Checked: ctest 49 / 49; the 20 browser tests; `xo-build --sweep`.

**Left in this ticket:** `std::deque<T>` (`outbound_q_`), which is also
guarded by the session's mutex, so it goes with `xo-websock/issues/15`.

## Related: transparent wrappers; CallbackId, 2026-10-06 -- umbrella `3b41edd7`, `21cc1825`

`Subscription::callback_id_` (`fn::CallbackId`) was the last
`member_as` that was not lock-guarded and not a regex or std::function.
- **Where the reflection lives.** xo-callback is header-only and does not
  depend on xo-reflect. RC pointed out that xo-webutil depends on both, so
  it hosts `reflect_callback_id()`; `websock_reflect_types` calls it. The
  private `id_` is reached through a new public member-pointer accessor,
  `CallbackIdImpl::id_address()`, which keeps xo-callback
  reflection-agnostic.
- **A struct was too busy.** First reflected as a struct with one member,
  `id`, each subscription grew a nested CallbackId box on the introspect
  page. RC judged it too busy, so it is reflected as an atomic instead.
- **New in xo-reflect: transparent wrappers.**
  - `WrapperTdx` (`wrapper/WrapperTdx.hpp/.cpp`): `mt_atomic`, with no
    children. `wrapped_td()`, and `wrapped_tp(obj)` reaches the wrapped
    member in place, through a `GeneralStructMemberAccessor`.
  - `WrapperReflector<T>` with `reflect_wrapped(memptr)` installs it,
    once.
  - `TypeDescrExtra::wrapper_info()` (`nullptr` by default) and
    `is_wrapper()`, forwarded by `TypeDescr`.
  - Tests (`WrapperReflector.test.cpp`): mt_atomic and `is_wrapper`; the
    wrapped type; the value reached in place; reflected once; a plain
    struct is not a wrapper.
- **printjson.** A wrapper with no printer prints as its wrapped value,
  `print(wrapped_tp(..))`; `printable` accepts a wrapper of a printable
  value. Test: a struct holding a wrapper prints the number.
- **websock.** `callback_id_` is reflected, and its `member_as` is gone.
  Golden diff against `d78751f4`: a reorder only. `callback_id_` is still
  `1`, metatype atomic, now ahead of `endpoint_`. The page shows the row
  `callback_id_: 1`, with no box.

Checked: ctest 49 / 49; the 20 browser tests; `xo-build --sweep`.
