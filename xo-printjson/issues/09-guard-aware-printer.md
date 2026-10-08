# 09 -- the generic printer takes reflection-declared guards

Status: open
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
