# 19 -- golden snapshot compares gcc's type spellings: clang CI red since it landed (2026-10-07)

Status: diagnosed
Type: bug / ci
Blocked by: none (`ci/issues/03` done 2026-10-11, umbrella `80e7f8f8`)
Raised: 2026-10-11 (RC noticed CI red on umbrella `357fc94c`)

`live-server-snapshot-matches-golden` (`xo-websock/utest/WebserverLive.test.cpp`)
compares the server snapshot, text for text, with
`xo-websock/utest/golden/server-snapshot.json` -- blessed under gcc.  The
snapshot carries `_canonical_type_` / `_short_type_` from
`TypeDescrBase::canonical_name_`, which `type_name<T>()` takes from
`__PRETTY_FUNCTION__`: the compiler's own spelling.  So under clang the test
can never pass.

## Diagnosis

`cmake-build (clang, clang++)` fails on exactly this assertion on every
commit from `21cc1825` (task 603, 2026-10-07) through `357fc94c` (task 612);
gcc and the nix smoke-test pass.  CI task list:

```bash
curl -s 'https://conybeare.us/git/api/v1/repos/roland/xo-umbrella2/actions/tasks?limit=20' \
  | python3 -c 'import json,sys; [print(t["id"],t["head_sha"][:8],t["name"],t["conclusion"]) for t in json.load(sys.stdin)["workflow_runs"]]'
```

Logs are on vpn1 only:
`zstd -dc /var/lib/forgejo/actions_log/roland/xo-umbrella2/64/612.log.zst`.
Extracting the snapshot from task 612's Catch2 expansion (re-joining its
line wraps) and comparing it with the golden as of `357fc94c`: **0
differences outside `_canonical_type_` / `_short_type_`; 44 inside**, e.g.

| gcc (golden) | clang (CI) |
|---|---|
| `std::__cxx11::basic_string<char>` | `std::basic_string<char>` |
| `long unsigned int` | `unsigned long` |
| `std::unordered_map<long unsigned int, ..., std::hash<..>, std::equal_to<..>, std::allocator<..> >` | `std::unordered_map<unsigned long, std::unique_ptr<..>>` |

**The test has never passed under clang.**  The golden held gcc spellings
from the commit that added it (`619860f3`, 2026-10-05 00:46); CI runs per
push, and the last green clang run (`e7c6a9b6`, 00:05) predates it, so the
first clang run to include it is `21cc1825` -- red.

```bash
git show 619860f3:xo-websock/utest/golden/server-snapshot.json | grep -c -E '__cxx11|long unsigned int'   # 21
git merge-base --is-ancestor 619860f3 e7c6a9b6 || echo "not in the last green run"
```

(A first reading, recorded per CONVENTIONS rule 6: "it began when std
members were reflected, 2026-10-06, bringing gcc spellings into a golden
that had none".  Plausible because those commits sit between the two CI
runs; wrong, because the first golden already carried them.  Also
unverified, and irrelevant to the fix: whether clang's missing `__cxx11`
means a different standard library or only clang's printing of inline
namespaces.)

**Collateral, worse than the red mark:** the cmake workflow
(`.forgejo/workflows/ci-cmake.yaml`) is one step per subsystem, and a
failed step ends the job.  Since 2026-10-07 every subsystem AFTER
xo-websock has not been built or tested under clang:

```bash
grep -oE '^      - name: build [a-z0-9-]+' .forgejo/workflows/ci-cmake.yaml \
  | awk '{print $4}' | awk '/^xo-websock$/{f=1;next} f'
```

(12 on 2026-10-11, xo-pywebsock through xo-pykalmanfilter.)

Not a source-link problem: the type -> source map keys on bare qualified
names, which gcc and clang spell alike (`issues/18`).

## Plan (RC, 2026-10-11): a golden per compiler

Type spellings belong to the compiler, and both compilers' spellings are
worth pinning: keep a golden for each.

- `golden/server-snapshot.json` -> `golden/server-snapshot.gcc.json`; add
  `golden/server-snapshot.clang.json`.  The test picks by `__clang__`,
  tested BEFORE `__GNUC__` (clang defines both); any other compiler fails,
  naming the file it would want.
- `XO_UPDATE_GOLDEN=1` rewrites the file for the compiler the test was
  built with.  The INFO lines on failure name that file.
- **Blessing the clang golden** needs CI's clang -- `.xo-backlog/ci/issues/03`
  (`xo-builder-shell`, the builder image's contents as a nix-shell; same
  store paths as CI, checked there).  Inside it, a separate tree configured
  with `CC=clang CXX=clang++` (e.g. `.build-clang`); build and run only
  `utest.websock.live`, which builds xo-websock and its dependencies.
- **First clang golden:** bless it that way, and compare with the snapshot
  CI itself printed (task 612, umbrella `357fc94c`; later umbrella commits
  touch no server code -- re-check before relying on it).  Identical =
  the shell reproduces CI's spellings, which `ci/issues/03` leaves
  unverified.

Was blocked by `.xo-backlog/ci/issues/03` (for blessing locally) -- done
2026-10-11, umbrella `80e7f8f8`.

**Superseded plan** (2026-10-11, same day; kept per CONVENTIONS rule 6):
one golden, blessed under gcc; under clang, blank `_canonical_type_` /
`_short_type_` in both actual and expected before comparing, and refuse
`XO_UPDATE_GOLDEN`.  It rejected a golden per compiler because "blessing
the clang one needs the CI's clang toolchain locally" -- true then,
answered by `ci/issues/03`; and it gave up checking clang's spellings at
all.  Still rejected: normalizing spellings (chases each compiler's
quirks: default template args, inline namespaces, `{anonymous}` vs
`(anonymous namespace)`).

**Done when:** `cmake-build (clang, clang++)` passes, all 72 steps.  Expect
the 12 subsystems it has skipped since 2026-10-07 to surface their own
clang failures, if any -- they get their own tickets.
