# 02 — DList::at() aborts instead of throwing on an out-of-range index

Status: diagnosed
Type: bug

`DList::at(ix)` with `ix >= size()` **SIGABRTs** in any build with assertions
enabled. Its `std::runtime_error` — and the message it carries — is unreachable
there.

Observed 2026-08-12 while adding coverage for that message, incidentally to
`.xo-backlog/xo-object2/issues/01-object2-free-of-indentlog.md`. Not caused by
that work: the defect is in the index walk, which it did not touch.

## The mechanism

`xo-object2/src/object2/DList.cpp`, `DList::at`:

```cpp
size_type ix = index;
const DList * l = this;

while (l->rest_ && (ix > 0)) {
    --ix;
    l = l->rest_;
}

if (ix > 0) {
    assert(l == nullptr);          // <-- always false when reached
    throw std::runtime_error(...); // <-- unreachable with assertions on
}
```

The loop's guard is `l->rest_`, so it stops with `l` pointing at the **last
node**, never at `nullptr`. Reaching `ix > 0` therefore means "ran out of list
while index remained" — with `l` non-null by construction. The assert states
the opposite of the invariant the loop establishes.

Reproduce:

```cpp
DList * l = DList::_cons(alloc, DInteger::box(alloc, 1), DList::_nil());
l->at(9);      // SIGABRT, not std::runtime_error
```

Confirmed by running exactly that inside `xo-object2/utest`: the test binary
died with `SIGABRT` before printing the caught message, while the sibling
`DArray::at(9)` in the same test threw and reported normally.

## Why it went unnoticed

```bash
grep -rn 'out-of-range\|REQUIRE_THROWS\|CHECK_THROWS' --include=*.test.cpp xo-object2/utest
```

returned nothing before 2026-08-12 — **neither** `DArray::at` nor `DList::at`
had any out-of-range coverage. `DArray::at` now does
(`DArray-at-out-of-range`); `DList::at` deliberately does **not**, because a
test for it would abort the whole binary rather than fail.

## Fixing it

Delete the assert, or replace it with one that states the real invariant:

```cpp
if (ix > 0) {
    assert(l && !l->rest_);        // ran out of list, l is the last node
    throw std::runtime_error(...);
}
```

Then add the `REQUIRE_THROWS_AS` coverage alongside `DArray-at-out-of-range` in
`xo-object2/utest/DArray.test.cpp`.

Worth checking at the same time, **unverified**: whether the boundary case
`ix == size()` behaves, and whether `at()` on the shared `s_null` sentinel
(`DList::_nil()` returns `&s_null`, not nullptr — see
`xo-object2/src/object2/DList.cpp:32`) takes a sensible path. The `size()` loop
just above `at()` tests `l != &s_null` explicitly; `at()` never mentions the
sentinel, which is a difference between two functions that walk the same
structure.

## Note for whoever fixes it

The unreachable `throw` builds its message with `xo::pp::tostr` /
`xo::pp::xtag` as of 2026-08-12. That conversion was verified against the
legacy vocabulary on `DArray::at`'s message, which **is** reachable and is
byte-identical across the two (colour escapes included). So the `DList` message
is expected to be correct once it can actually run — but it has never been
observed, and should be when the assert goes.
