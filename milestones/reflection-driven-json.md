# reflection-driven-json -- one generic struct printer, driven by reflection

Status: open
Type: milestone

Retire the per-type json printers in favour of one generic struct printer in
xo-printjson, driven by xo-reflect's description of each type. Bespoke struct
printers may remain to finesse exceptions, but each is a distasteful
concession to expedience: the goal is to need none (RC, 2026-10-05).

Follows `.xo-backlog/xo-printjson/issues/02`, which made every object print
once (`"_id_"` / `{"_ref_": n}`) and threaded a `JsonPrintState` through every
printer, using an object writer (`JsonObject`) that both generic and bespoke
printers share.

## Where it starts (2026-10-05, umbrella `e7c6a9b6`)

- **xo-reflect can only describe data members.** `StructReflector` takes
  member pointers (`reflect_member(name, &T::m_)`,
  `xo-reflect/include/xo/reflect/StructReflector.hpp:56-69`). It has no
  getters, computed properties or per-member attributes.
- **Several types are opaque leaves.** The metatypes are atomic, pointer,
  vector, struct and function (`Metatype.hpp:9`). `std::atomic`, the maps,
  `std::function`, enums, `std::deque` and `std::unique_ptr` have no
  `EstablishTdx` specialisation, so they fall to `AtomicTdx`.
- **The websock types are mostly reflected with no members.** Only
  `WebserverConfig` and `WebsocketSinkImpl` reflect any.
- **Each websock printer does four jobs that reflection can't yet:**
  1. placement, in full here or a ref (`member` vs `member_ref`);
  2. safe reads: an atomic's `load()`, or copies taken under a mutex (router,
     url router, session table, session);
  3. summaries of types reflection can't see (enum, regex, function, deque,
     `unique_ptr`, `CallbackId`);
  4. view-model top-level keys (`listen_port`, `state`, `endpoints[]`,
     `sessions[]`, `subscriptions[]`, `sink`, `sender`, `receiver`,
     `session_id`, `sub_id`, `stream`, `kind`, `stem`, `pattern`, `open`).

  introspect reads only the keys in job 4; everything else it reads from
  `_members_`.

## Decisions (RC, 2026-10-05)

- **Wire format: members-style for every reflected struct:**
  `{"_name_", type keys, "_id_", "_members_": [...]}`, the shape
  `JsonMembers` writes now. A struct may then have members literally named
  `_id_` or `_ref_` without ambiguity.

  The flat format (members as top-level keys: generic structs today, and
  the flywheel frame, a declared wire contract in
  `xo-object2/utest/flywheel_frame.test.cpp`) is likely an atavism that
  needs upgrading. Json conversion is a kind of pretty-printing, so in the
  long run a choice like this may be configurable at runtime.
- **Placement: first encounter, for now.** The generic printer prints an
  object in full where the traversal first reaches it, and as a ref
  afterwards. Explicit ownership (an attribute on a reflected member) was
  considered and deferred, because it makes printing unreliable: a ref is
  safe only if its target prints in full somewhere in the same output, and
  an owner-only rule leaves an object unprinted when the owner's path is
  not traversed. It is easy to add later if placement matters.

  Known consequence: the server's `pjson_` will print inside the first sink
  again, once the server's, router's and sink's printers are generic.
- **Locking: printjson does not lock.** Threading is handled by locking above
  the scope of a printjson call: the caller holds what it needs for the
  whole print.
- **Maps: a new xo-reflect metatype,** on its own ticket. Custom printers in
  the meantime.

## Done when

- every websock struct prints through the generic printer, with no bespoke
  printer left, or each one that remains is recorded as an exception;
- introspect reads only `_members_`, and the view-model top-level keys are
  gone;
- the flywheel frame is upgraded to members-style, or recorded as the one
  flat exception along with its consumer;
- `JsonMembers::member_as` / `member_ref` survive only at recorded
  exceptions.
