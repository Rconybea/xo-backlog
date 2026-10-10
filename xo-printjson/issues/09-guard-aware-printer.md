# 09 -- the generic printer takes reflection-declared guards

Status: done 2026-10-10 -- umbrella `3bbda0b6`
Type: feature
Milestone: reflection-driven-json
Blocked by: `.xo-backlog/xo-printjson/issues/08`, `.xo-backlog/xo-reflect/issues/08`

RC (2026-10-08), revising the milestone's 2026-10-05 "printjson does not
lock": the generic struct printer acquires locks, but only those xo-reflect
declares (`.guarded_by`, `xo-reflect/issues/08`).  printjson never names a
mutex.  A separate xo-reflect walker, with printjson as one visitor of it,
was considered and deferred until a second consumer (serializer, GC-style
scan) appears.

## Behaviour

- `print_generic_struct` writes `_members_` grouped by guard, through the
  xo-reflect helper: unguarded members, then each guard's group with that
  guard held.  Each group is a consistent snapshot.  `_members_` order
  changes from declaration order to guard-group order; introspect reads
  members by name.
- While a guard is held, descend through owning and shared edges -- child
  guards nest inside, in ownership order.  Holding it is what keeps a
  guarded owning edge's target alive (e.g. a `unique_ptr` in a guarded map).
  Borrowed edges acquire nothing (`issues/08`).
- Mode, on `JsonPrintState`: **blocking** (default) or **try**.  In try mode
  a busy guard's members print as `{"_locked_": true}`.

## The rule for callers

The calling thread must hold **no** guard of the structure it prints.
`std::mutex` is not recursive, and `try_lock` on a mutex the caller owns is
undefined -- try mode does not rescue it.  Document this on the print entry
points.  Worth an XO_PRINTJSON_REENTRY_CHECK-style debug check if a cheap
one exists (unverified).

**Done when:** a utest prints a two-guard struct while another thread
mutates it (no race under TSan, if the build supports it -- unverified),
try mode prints `_locked_` for a held guard, and the caller rule is
documented.

## As built (umbrella `3bbda0b6`)

- `JsonMembers::reflected_members` walks members through
  `reflect::visit_members_guarded` (unguarded first, then each guard's
  group, held); each entry by `write_reflected()`.  So every caller gets
  guards -- `print_generic_struct` and the bespoke printers that call
  `reflected_members` alike.
- Mode: `PrintJson::guard_mode()` / `assign_guard_mode()`, default
  `blocking`, copied into `JsonPrintState` like `max_depth`.
- Try mode: a busy guard's members are entries without a value,
  `{"_name_", type keys, "_metatype_", "_locked_": true}` -- an entry-level
  key beside `_error_`, not `{"_locked_": true}` as the value (RC
  2026-10-10), so `_value_` always means a value.
- Caller rule (hold no guard): documented on `PrintJson`, `JsonPrintState`,
  `reflected_members`; not checked -- no cheap check exists (try_lock by the
  owner is undefined; the print cannot see the caller's locks).
  `std::recursive_mutex` considered (RC asked) and not adopted: websock's
  `mutex_` pairs with a `std::condition_variable`, recursion hides
  re-entrancy bugs, and it makes reading mid-critical-section legal, not
  safe.  An owner-tracking debug mutex is the route if a check is wanted.
- TSan is not set up in this tree; instead
  `print-json-guard-consistent-under-writes` (writer keeps `a_ == b_`),
  falsified: with `guarded_by` removed it fails with torn reads.
- Tests: `xo-printjson/utest/PrintJsonGuard.test.cpp`.
