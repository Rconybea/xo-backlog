# Backlog conventions

How tickets in this repo should be written, and why.

These exist because tickets are read months later, detached from the
conversation that produced them, and are then treated as established fact. A
plausible-sounding claim in a ticket has more authority than it earned.

## The failure this is guarding against

Three times on 2026-08-08, a **characterization** was asserted on the strength
of an **enumeration**:

| enumeration (measured) | characterization (asserted, wrong) |
|---|---|
| "7 subsystems still declare an indentlog dep" | "the 7 are the printable2 stack" — xo-expression isn't |
| "the library and its examples have different guards" | "the failure is a guard mismatch" — it was an unused dependency |
| "`printer` appears in RealizationSource.test.cpp" | "xo-process is blocked on `printer`" — both hits were dead code |

The pattern in all three: **a claim about *why*, built from evidence about
*what*.** Presence data promoted to a causal story. Each was cheap to check and
none was checked. Two of the three were written into tickets, where the imgui
one came with three proposed remedies, two of which would have made things
worse.

Note the asymmetry that let it happen: the same session ran 61-subsystem builds,
34-suite test sweeps, `.o.d` header audits and nix builds — heavy verification
aimed at code already written. The single sentence deciding *what to do next*
got none. **The load-bearing claim is usually a sentence, not a diff.**

## Rules

### 1. Every structural claim carries its command, or is marked unverified

A claim about *why* something is the way it is, or about which things belong to
a group, needs the command that established it — inline, runnable, next to the
claim. If it hasn't been run, say so in the text: "unverified", "worth
confirming".

Good (from `generated-find-dependency/issues/02`):

```bash
P=$(nix-build ci.nix -A xo-process --no-out-link)
grep -n find_dependency          $P/lib/cmake/process/processConfig.cmake
grep -n INTERFACE_LINK_LIBRARIES $P/lib/cmake/process/processTargets.cmake
```

As of 2026-08-08, 20 of 30 tickets carried no command at all — and the analysis
tickets, the ones most likely to be read as fact, were the least grounded.

### 2. Re-derive labels; never carry them

If a sentence names a group — "the facet cluster", "the stragglers", "the deck"
— recompute membership in the same breath. Group labels are accurate when
coined and rot silently as work lands. "What's left is the facet cluster" was
true at 11 subsystems and false at 7; nothing re-checked it because it had
become a phrase rather than a claim.

### 3. Falsify the load-bearing sentence

Before a ticket recommends an action, ask: **what one command would show this is
false?** Then run it. This is the rule that would have caught all three failures
above, and it costs seconds.

For dependency claims the command now exists — see below.

### 4. Status carries confidence

`Status:` distinguishes how well the diagnosis is established, not just whether
work remains:

- `hypothesised` — a story that fits the evidence, not yet confirmed
- `diagnosed` — root cause confirmed by a command recorded in the ticket
- `open` / `fixed <date>` / `resolved <date>` — as before, for work state

`xo-imgui/issues/01` read `Status: open` above a confidently wrong diagnosis.
`hypothesised` would have flagged it as needing the check.

### 5. Record corrected diagnoses, don't overwrite them

When a ticket's diagnosis turns out wrong, say so in the ticket and keep the
wrong reading with a note on why it was plausible. The next person will reach
for it too. `xo-imgui/issues/01` does this.

## Checking dependency claims: `xo-deps`

Most structural claims in this repo are about the subsystem graph, so that case
has direct support. `xo-deps` gained two query modes on 2026-08-08 for exactly
this purpose (`xo-cmake/bin/xo-deps.in`):

```bash
# does X depend on Y, directly or transitively?
# prints the shortest path as evidence; exit 0 = yes, 1 = no, 2 = bad name
xo-deps --why=xo-tokenizer2:xo-printable2
#   xo-tokenizer2 -> xo-stringtable2 -> xo-printable2

xo-deps --why=xo-expression:xo-printable2      # prints nothing, exit 1

# blast radius: everything a rework of Y could touch
xo-deps --users-of=xo-printable2 --format=names -q

# upstream closure: everything X needs
xo-deps --deps-of=xo-expression --format=names -q
```

`--format=names` writes plain names to stdout with the summary on stderr, so it
pipes; `-q` drops the summary entirely. Neither mode needs graphviz.

Exit 2 (unknown subsystem) is deliberately distinct from exit 1 (no path), so a
typo cannot silently read as "not a dependency".

The claim that started all this is now one loop:

```bash
for s in $(xo-deps --users-of=xo-indentlog --format=names -q); do
    xo-deps --why=$s:xo-printable2 -q >/dev/null \
        && echo "$s in cluster" || echo "$s INDEPENDENT"
done
```

**Use `-q`, not `2>/dev/null`.** Redirecting stderr also hides
`xo-deps: no such subsystem`, so a mistyped name would come back as
"INDEPENDENT" — a falsification check silently confirming the claim it exists to
refute. `-q` suppresses only the summary; errors still surface, and exit 2 still
distinguishes them from a real "no".

**Bound blast-radius claims with `--users-of`.** "Reworking xo-printable2 affects
X" is only checkable if the reader can see the whole set; `--users-of=Y
--format=names` is that set, and it is 11 subsystems, not "everything".

## Verifying a change in this tree

The recipe below was re-derived from scratch several times on 2026-08-08, and
got it wrong twice — once by omitting examples, once by discarding a build
failure. Written down so the next pass starts from the working version.

```bash
# 61 buildable subsystems: xo-equable2 and xo-hashable2 are empty placeholder
# subrepos (README + .gitrepo, no CMakeLists) and abort `xo-build --all`
SUBS=$(xo-build --list | tr ' ' '\n' | grep '^xo-' \
       | while read s; do [ -f "$s/CMakeLists.txt" ] && echo "$s"; done)

xo-build -q --configure --with-utests --with-examples --build --install $SUBS
```

Four things that recipe encodes:

- **`--with-examples`.** Utests alone miss a whole class of breakage: examples
  are compiled by nix but not by a default umbrella build, so a broken example
  passes locally and fails in CI. That is how the xo-tokenizer `tokenrepl`
  regression escaped, and how the pre-existing xo-imgui guard bug stayed hidden.
- **`-q`.** Quiet on success, loud on failure — 61 lines instead of 3060, and
  unlike `>/dev/null 2>&1` it cannot hide an error. **Never redirect a build to
  /dev/null**: it discards the diagnostics too, so a failing build looks
  identical to a passing one, and any `ctest` that follows silently runs the
  *previous* binary. That cost a long debugging detour on 2026-08-08 —
  symptoms looked like a library bug; the cause was a compile error in a test
  file that had been redirected away.
- **Pass the list to `xo-build`; do not loop over it.** It takes a subsystem
  list and already prints one status line per subsystem — that is what `-q`
  added (`.xo-backlog/xo-cmake/issues/01`). A shell loop only duplicates that
  line, and the duplication is what tempts you into `>/dev/null` to tidy it up,
  which is how the rule above gets broken by someone who already knows it.
  That exact sequence happened on 2026-08-08; see
  `.xo-backlog/xo-cmake/issues/03`. `ctest` below is the exception, and only
  because `--utest` cannot yet survey past a failure.
- **`--install`.** Subsystems find each other through installed configs; a
  build without install validates less than it appears to.
- **The subsystem list, not `--all`.** `--all` aborts on the two placeholders.

### Then the checks that actually catch things

```bash
# tests: expect 34 passing, 1 failing (xo-jit / machpipeline.fptr, ticketed)
#
# NB the loop is NOT the pattern to copy -- `xo-build -q --utest $SUBS` does
# this properly (one status line per subsystem, loud on failure) EXCEPT that it
# stops at the first failing subsystem, and xo-jit fails permanently, so a
# sweep would silently leave everything after it untested.  See
# .xo-backlog/xo-cmake/issues/03; when that lands, delete this loop.
for s in $SUBS; do ctest --test-dir $s/.build; done

# nix: the ONLY check that exercises an installed package config as a real
# consumer would.  Caught -lindentlog, the xo-tokenizer example, and
# xo-reader's empty propagatedBuildInputs -- none of which the umbrella saw.
nix-build ci.nix -A <subsystem> --no-out-link

# "is this dependency really gone": ask the preprocessor, not grep.  NB match
# 'xo/indentlog/' WITH the slash -- bare 'indentlog' also matches indentlog2.
find <sub>/.build -name '*.o.d' | xargs grep -lE 'xo/indentlog/'

# exported config must resolve every name it exports
grep -n find_dependency          <prefix>/lib/cmake/<n>/<n>Config.cmake
grep -n INTERFACE_LINK_LIBRARIES <prefix>/lib/cmake/<n>/<n>Targets.cmake
```

**Stale build artifacts lie.** `.o.d` files survive source renames, so a
`grep -l` over them can report a dependency from a file that no longer exists
(`GlobalEnv.cpp` after the rename to `GlobalSymtab.cpp`). If a result looks
impossible, `rm -rf <sub>/.build` and rebuild before believing it.

## Ticket shape

No rigid template — existing tickets vary usefully. But most should have:

- `Status:` and `Type:` lines (see rule 4)
- what was measured, with commands and a date
- what is inferred but unverified, said plainly
- **Files:** the concrete paths, with line numbers where they matter
- **Done when:** what would close it

Sections like `## Why it went unnoticed` and `## Notes` have proven the most
valuable to re-readers — they carry the reasoning that the diff cannot.
