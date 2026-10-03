# 03 — `TypeDescr::short_name()` for template types; `rp<T>`

Status: done 2026-10-03 -- umbrella `9e251d5a`; CI fixes `af3fe344`, `b3ac8417`
Type: feature

`TypeDescr` carries two names: `canonical_name()` (full, unique) and
`short_name()`, so display can suppress excessive detail (RC). For template
types the short name kept nearly all of the detail.

## Before

`unqualified_name()` (`xo-reflect/src/reflect/TypeDescr.cpp`) dropped only the
qualifier before the first `<`; template arguments kept theirs. `std::pair`
was special-cased to bare `pair`. The comment on `short_name_` said "just after
last ':'", which the code did not do.

| canonical name | short name |
|---|---|
| `xo::ref::intrusive_ptr<xo::web::DynamicEndpoint>` | `intrusive_ptr<xo::web::DynamicEndpoint>` |
| `std::unordered_map<std::__cxx11::basic_string<char>, …, std::hash<…>, std::equal_to<…>, std::allocator<…> >` | all of it but the leading `std::` |
| `std::pair<int, double>` | `pair` |

The introspect page (`.xo-backlog/xo-websock/issues/13`) worked around this in
JS (`short_type()` in `introspect.js`).

## After

`TypeDescrBase::make_short_name(std::string_view canonical)` (public static,
so it can be tested on strings) builds `short_name_`, in this order:

1. `xo::ref::intrusive_ptr<` -> `rp<` -- the shorthand xo code uses (RC).
   Matched before qualifiers go, so boost's or anyone else's `intrusive_ptr`
   keeps its name. `rp` is `xo::rp<T>`, alias for `ref::intrusive_ptr<T>`:
   `grep -n "using rp" xo-refcnt/include/xo/refcnt/Refcounted.hpp`.
2. strip every namespace qualifier, template arguments' included, and both
   spellings of the anonymous namespace (`{anonymous}::`,
   `(anonymous namespace)::`). A member pointer keeps its class (`Foo::*`).
3. no space before `>`.
4. drop the standard library's default template arguments gcc spells out:
   `, allocator<..>`, `, char_traits<..>`, `, default_delete<..>`,
   `, hash<..>`, `, equal_to<..>`, `, less<..>` (never a first argument). An
   explicit other one (e.g. `greater<int>`) stays.
5. `basic_string<char>` -> `string`.

e.g. `rp<DynamicEndpoint>`, `unordered_map<string, rp<DynamicEndpoint>>`,
`pair<int, double>`.

`short_name_` became `std::string` (the result is no longer a substring of
`canonical_name_`); `short_name()` returns `std::string_view` by value (was
`std::string_view const &`). Every caller compiled unchanged (the sweep below).

Display users of `short_name()` (all shorter now, none broken):

```bash
grep -rn "short_name()" --include=*.cpp --include=*.hpp xo-*/ | grep -v '/\.build/'
```

-- xo-printjson's `"_name_"` (`PrintJson.cpp`), log tags in xo-jit,
xo-object, xo-expression; tests in xo-ratio, xo-object compare plain names,
unchanged.

## Verified

- `xo-reflect/utest/ShortName.test.cpp`: the rules on fixed strings (gcc
  spellings, clang's anonymous namespace, boost's intrusive_ptr, a member
  pointer) and on real types as this compiler spells them (`rp<Probe>`,
  `vector<string>`, `pair<int, double>`). 20 assertions.
- ctest 49/49; `xo-build --sweep -j 8`: 73 attempted, 73 ok (build), 47 ok +
  26 with no tests (utest), `--sweep ok`; the 11 introspect browser tests
  (scratch, ticket xo-websock/13) pass.

## Open: short names are not unique -- xo-jit names LLVM structs with them

`a::Foo` and `b::Foo` always shared a short name; now `Foo<a::X>` and
`Foo<b::X>` do too. Display does not care. xo-jit does: it passes
`short_name()` as the name of an LLVM struct type.

```bash
grep -n "short_name()" xo-jit/src/jit/type2llvm.cpp    # struct_name = ..., then StructType::create(.., StringRef(struct_name), ..)
```

Its comment there says names "within an llvmcontext, must be unique".
UNVERIFIED what a clash does: my understanding is that LLVM's
`StructType::create` renames a taken name by appending a suffix (`Foo.0`), in
which case a clash is cosmetic (IR readability) rather than wrong. Worth
confirming before choosing a fix; if it matters, the obvious fix is to name
LLVM structs from `canonical_name()` (unique), keeping `short_name()` for
display.

## CI: the test spelled gcc's anonymous namespace, 2026-10-03

GitHub `cmake-docker`, clang job, red from `9e251d5a` on (gcc job green):

```bash
gh run view 37129032964 --log-failed | grep -A4 'ShortName.test.cpp:79'
#   REQUIRE( Reflect::require<ShortNameProbe>()->canonical_name() == "xo::ut::{anonymous}::ShortNameProbe" )
# with expansion:
#   "xo::ut::(anonymous namespace)::ShortNameProbe" == "xo::ut::{anonymous}::ShortNameProbe"
```

A test bug, not a `make_short_name()` one: the other 19 assertions passed
under clang, `(anonymous namespace)::` case included. Fix: build the
expected canonical name from `type_name<ShortNameProbe>()` rather than
spell it, as the other xo tests do. gcc: utest.reflect 259 assertions pass.
Not run under clang locally (the local clang++, used for the source map,
cannot link: cmake's compiler check fails) -- the CI clang job is the check.

The same clang job had been failing earlier on an unrelated link error in
xo-object2 (`RAllocator<..>::barrier_assign` undefined, e.g. run
37098295886); CI builds xo-reflect first, so this test's failure has been
masking it since `9e251d5a`. Expect it next. Not diagnosed yet.

## CI again: pointer spacing differs by compiler, 2026-10-03

Umbrella `b3ac8417`.

With xo-reflect green under clang (`af3fe344`), the clang job reached
xo-printjson and failed there (run 37134962597):

```bash
gh run view 37134962597 --log-failed | grep -A6 'with expansion' | cut -c1-300
#  "_canonical_type_": "xo::ut::(anonymous namespace)::JmUnreflected *", "_short_type_": "JmUnreflected *"
```

`JsonMembers.test.cpp` (from `.xo-backlog/xo-printjson/issues/05`) spelled
the short type `"JmUnreflected*"` -- gcc's. Clang writes a space before `*`
and `&` (`Foo *`, `const Foo &`). Two changes (RC: both):

1. `make_short_name()` now drops whitespace before `*` and `&` as well as
   `>`, so a short name reads the same from either compiler (`Foo*`,
   `const Foo&`, `Foo&&`). ShortName.test gains both spellings of each, a
   vector of pointers and a function taking a reference (27 assertions).
   The canonical name is untouched (it stays the compiler's spelling: the
   key for lookups and the source map).
2. JsonMembers.test builds both expected short types with
   `make_short_name(type_of<..>())`, as its `entry()` helper does, rather
   than spelling them.

Scan for other compiler-specific spellings in today's tests:
`grep -rnE '\{anonymous\}|[A-Za-z_>]\*"|[A-Za-z_>]&"|> >"' --include=*.test.cpp xo-*/utest`
-- only comments and typeseq.test.cpp, which handles both spellings
deliberately. The remaining literal type names in websock / reflect tests
are named, non-anonymous, pointer-free, so spelled alike.

gcc: build, ctest 49/49, utest.reflect short_name 27, utest.printjson 62,
the 12 introspect browser tests. `xo-build --sweep -j 8`: 73 attempted, 73
ok (build); 47 ok + 26 with no tests (utest); `--sweep ok`. Clang: the CI job.
