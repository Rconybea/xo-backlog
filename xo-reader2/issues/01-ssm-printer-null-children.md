# 01 — SSM printers abort or throw on children that are null in reachable states

Status: diagnosed
Type: bug
Progress: grep -rlE '\.variant<APrintable|obj<APrintable,D[A-Za-z]+> [a-z_]+\(' xo-reader2/src --include=*.cpp | grep -v '/\.build/' | wc -l

Several xo-reader2 syntax state machines hold a child pointer that is null
until a particular token arrives, and their printers dereference it
unconditionally. Printing such an ssm in that window does not produce a
degraded rendering — it **terminates**, by one of three different mechanisms.

Found incidentally while converting these printers to ppsink
(`.xo-backlog/xo-printable2/issues/01`); every case below was **observed**, not
read off the source.

## The four cases, and their three failure modes

| ssm | member | state where null | printing it |
|---|---|---|---|
| `DExpectQDictSsm` | `dict_` | before `on_leftbrace_token` | **SIGABRT** |
| `DExpectQListSsm` | `start_` | before the first element | **throws** |
| `DExpectQArraySsm` | `array_` | before the first element | **throws** |
| `DDefineSsm` | — (`defstate_`) | `def_0`, i.e. as `_make` leaves it | **SIGABRT** (assert) |

Reproduced 2026-08-11 by rendering a bare `_make()`d instance of each:

```
DExpectQListSsm   FacetRegistry::variant failed :AFrom.tname xo::mm::AGCObject
                  :ATo.tname xo::print::APrintable :DRepr 8
DExpectQArraySsm  ... :DRepr 7
DExpectQDictSsm   fatal: attempt to call uninitialized IPrintable_Any method
                  terminate called without an active exception -> SIGABRT
DDefineSsm        DDefineSsm.cpp:411: get_expect_str(): Assertion `false' failed
```

`DDefineSsm` is a different mechanism from the other three — no child is
dereferenced; `get_expect_str()` simply has no answer for `def_0` and asserts.
Grouped here because the symptom and the trigger are identical from a caller's
point of view: **an ssm in the state its own `_make` produces cannot be
printed.**

## Why two of them abort and two throw

The failure mode is decided entirely by **how the printer obtains the
`APrintable` handle**, not by anything about the ssm:

```cpp
/* DExpectQList/QArraySsm -- registry lookup: fails LOUDLY, catchably */
auto list_pr = FacetRegistry::instance().variant<APrintable,AGCObject>(list);

/* DExpectQDictSsm -- direct construction: no lookup, no check */
obj<APrintable,DDictionary> dict_pr(dict_);
```

A directly-constructed `obj<A,DRepr>` over a null pointer is indistinguishable
from a live one until something calls through it, at which point it hits an
uninitialized vtable slot and `abort()`s. The registry form at least knows the
lookup failed.

This is the same defect described in
`.xo-backlog/xo-expression2/issues/02-dtypename-null-type-aborts.md`, whose
"wider question" section asks whether `to_facet<A>()` on an empty `obj<>`
should reject rather than return something that aborts on use. **These four
cases are the evidence that it is a pattern and not a one-off** — and that the
mechanism, not the printer, is what needs fixing.

## A fifth site, on an error path

```bash
grep -n 'obj<APrintable,DArray> arglist_pr' xo-reader2/src/reader2/ParserStateMachine.cpp
```

`ParserStateMachine::illegal_parsed_formal_arglist` builds `obj<APrintable,DArray>
arglist_pr(arglist)` and prints it into an error message. Same direct-construction
hazard, on the path that runs **when something has already gone wrong** — so it
would abort while trying to report a parse error. Worth fixing first if these
are ever done piecemeal.

## A sixth and seventh site, in xo-interpreter2 — a different mechanism (2026-08-12)

Found while converting `DLocalEnv`'s printer. These are **not** the
empty-`obj<>` mechanism above; they are a plainer bug, a raw pointer
dereferenced with no guard:

```bash
grep -n "args_->size()" xo-interpreter2/src/interpreter2/DLocalEnv.cpp \
                        xo-interpreter2/src/interpreter2/DVsmApplyFrame.cpp
```

`DLocalEnv::pretty` renders `field("n_args", args_->size())`, and
`DLocalEnv::_make` asserts `symtab` but **not** `args`
(`xo-interpreter2/src/interpreter2/DLocalEnv.cpp:31`). So a null `args_`
segfaults — on both protocols, and before the ppsink conversion as much as
after. `DVsmApplyFrame`'s printer does the same thing with its own `args_`;
whether that one can actually be null is **unverified** and should be checked
when it converts.

Filed here rather than in a new ticket because the fix shapes below apply
unchanged, even though the ticket's title says SSM. Worth noting the contrast
for whoever fixes these: the reader2 cases need the *facet* mechanism decided
(see "Fixing it"), whereas these two need only a null check or an assert in the
factory — they are separable, and cheaper.

## Reachability: latent, and the refactor makes it worse

**Not reachable at token boundaries.** Across five token streams
(`#q { ( 1 2 ) }`, `#q { [ 1, 2 ] }`, `#q { { a : 1 } }`, a `def`, and a
`lambda`), the whole parser — which renders `:stack`, walking every ssm — was
rendered after every token, ~40 renderings, and none threw or aborted. By the
time `on_token` returns, every ssm on the stack has a populated child.

**Reachable during a token.** `DExpectQLiteralSsm` pops itself, starts the
specialised ssm, and re-feeds the token:

```bash
grep -n -A2 'DExpectQ\(List\|Array\|Dict\)Ssm::start(p_psm);' \
    xo-reader2/src/reader2/DExpectQLiteralSsm.cpp
```

Between `start(p_psm)` and `p_psm->on_token(tk)` the stack holds a
freshly-made, unpopulated ssm. Anything printing the parser or the stack from
inside an `on_token` handler — a debugger, a new `log && log(xtag("parser",
this))`, an exception path that dumps state — lands in the window.

**And the ppsink migration removes the accidental protection.** Today
`DSchematikaParser::pretty` and `ParserStack::pretty` are phase-B stubs, so the
ppsink path renders `STUB:DSchematikaParser` and never walks the stack at all.
Once those two convert (the last of that migration), printing a parser on the
ppsink path will walk every ssm, exactly as the legacy path already does.
`DSchematikaParser::include_token` already logs `xtag("parser", this)` under
its debug flag:

```bash
grep -n 'xtag("parser", this)' xo-reader2/src/reader2/DSchematikaParser.cpp
```

That is the reason this ticket is worth doing **after** the refactor rather
than during it: the refactor is a pure-rendering-unchanged contract, so it must
reproduce these faults rather than fix them, and it is what widens the exposure.

## A wrong reading, kept

While converting `DExpectFormalArglistSsm` (2026-08-11) it was claimed to be a
fifth instance — "`_make` leaves `argl_ == nullptr` and the printer calls
`argl_->size()`". **It is not.** `_make` allocates the array:

```bash
sed -n '50,70p' xo-reader2/src/reader2/DExpectFormalArglistSsm.cpp
#   DArray * argl = DArray::_empty(mm, 8);
#   return new (mem) DExpectFormalArglistSsm(argl);
```

and a bare instance renders `<DExpectFormalArglistSsm :fastate argl_0 :expect
leftparen :n_args 0>` on both protocols. Plausible because the header does
declare `DArray * argl_ = nullptr;`, and because three genuine instances had
just been found — the default member initializer was read as the post-`_make`
state without checking the constructor that overrides it. The general lesson is
rule 3's: the load-bearing sentence was one line of a header, and it cost
nothing to check.

## Fixing it

Two shapes, and they are not alternatives — the second is the real fix:

1. **Per printer:** `try_variant` plus a placeholder for the registry cases; an
   explicit null test before constructing the handle for the direct cases; and
   for `DDefineSsm`, a `get_expect_str()` that returns something for `def_0`
   instead of asserting. Small, local, and testable — a bare `_make()`d
   instance of every ssm renders.
2. **At the facet layer:** make a directly-constructed `obj<A,DRepr>` over a
   null pointer, or `to_facet<A>()` on an empty `obj<>`, fail where the mistake
   is made rather than where the handle is used. That removes the whole class
   and is tracked at `.xo-backlog/xo-expression2/issues/02`.

Doing (1) alone is defensible here — a debug printer that takes the process
down is bad regardless — but it should not be mistaken for closing the class.

**Files:**
- `xo-reader2/src/reader2/DExpectQDictSsm.cpp` — direct construction, aborts
- `xo-reader2/src/reader2/DExpectQListSsm.cpp` — registry lookup, throws
- `xo-reader2/src/reader2/DExpectQArraySsm.cpp` — registry lookup, throws
- `xo-reader2/src/reader2/DDefineSsm.cpp` — `get_expect_str()` asserts on `def_0`
- `xo-reader2/src/reader2/ParserStateMachine.cpp` — the error-path site
- `.xo-backlog/xo-expression2/issues/02-dtypename-null-type-aborts.md` — the
  facet-layer question these are instances of
- `.xo-backlog/xo-printable2/issues/01-aprintable-pretty-ppsink.md` — the
  refactor that must land first

**Done when:**
- a bare `_make()`d instance of every xo-reader2 ssm renders without throwing
  or aborting, on the surviving (`PpSink`) protocol
- there is a test that renders each one, so a new ssm cannot reintroduce it
- the error-path site in `ParserStateMachine` is covered too
- a decision is recorded on whether (2) is being taken, or (1) is the end of it

## Notes

None of this was found by reading the printers. It was found by trying to write
a test that renders each one, which is worth remembering as a technique: the
states an object can be printed in are not obvious from its printer, and a
printer is the one method nobody writes tests for.
