# 02 — CI checkout built `main`'s tip at job start, not the run's commit

Status: fix `f2938083` broke checkout (dubious ownership); its fix not yet committed
Type: bug

## Symptom

GitHub `cmake-docker` run 37138286651, labelled `dcfe96bd`, went green under
clang -- yet `dcfe96bd`'s JsonMembers.test.cpp still spelled a short type the
gcc way, which clang fails (`.xo-backlog/xo-reflect/issues/03`, fixed in
`b3ac8417`):

```bash
git show dcfe96bd:xo-printjson/utest/JsonMembers.test.cpp | grep -c 'JmUnreflected\*"'   # 1
git show b3ac8417:xo-printjson/utest/JsonMembers.test.cpp | grep -c 'JmUnreflected\*"'   # 0
gh run view --job 111247224028 --log | grep 'utest.printjson'                             # Passed
```

## Cause

Every workflow checked out with

```sh
git clone --quiet --depth=1 $GITHUB_SERVER_URL/<repo>.git .
```

-- `main`'s tip when the JOB STARTS, not the commit that triggered the run.
That job started 17:09:12Z, a minute after `b3ac8417` was pushed (its run
created 17:08:23Z), so the run labelled `dcfe96bd` built `b3ac8417`:

```bash
gh run list --limit 8 --json headSha,createdAt --jq '.[] | "\(.headSha[0:8]) \(.createdAt)"'
gh run view --job 111247224028 --log | grep -m1 'checkout'
```

With runs queued -- as all afternoon of 2026-10-03 -- a red result can blame
the wrong commit and a green one vouch for code it never built. (The clang
results for `af3fe344` / `ccf65117` / `b3ac8417` read earlier that day are
therefore uncertain as to which commit they describe; the conclusion -- clang
green from `b3ac8417` on -- stands, from this run.)

All three templates had it:

```bash
grep -n 'git clone' .github/workflows/*.j2 .forgejo/workflows/*.j2   # now: none
```

## Fix

In `.github/workflows/ci-cmake.yaml.j2`, `.forgejo/workflows/ci-cmake.yaml.j2`,
`.forgejo/workflows/ci.yaml.j2`, fetch exactly the run's commit:

```sh
git init --quiet .
git remote add origin <url>
git fetch --quiet --depth=1 origin "$GITHUB_SHA"
git checkout --quiet FETCH_HEAD
echo "checked out $(git rev-parse HEAD)"
```

-- the log now names the commit built. Workflows regenerated with
`cmake --build .build --target xo-gen-ci` (which, before the edit, reproduced
the committed workflows with no diff).

## Verified

- Fetch-by-SHA of a NON-tip commit (`dcfe96bd`) with the step's own
  commands, against both servers: github.com/Rconybea/xo-umbrella2 and
  conybeare.us/git/roland/xo-umbrella2 -- both check out `dcfe96bd...`.
- The three regenerated workflows parse (PyYAML); each has one checkout
  step, using `$GITHUB_SHA`, with no `git clone`.
- Not yet run in CI: the first run after this lands is the check -- its
  checkout step should print the run's own `headSha`.

## The fix broke checkout: "dubious ownership", 2026-10-03

`f2938083`'s own run (37141547091) failed at checkout, both jobs:

```bash
gh run view 37141547091 --log-failed | grep -A3 'dubious'
# fatal: detected dubious ownership in repository at '/__w/xo-umbrella2/xo-umbrella2'
#         git config --global --add safe.directory /__w/xo-umbrella2/xo-umbrella2
```

The job runs in the docker-xo-builder container as root; the mounted
workspace belongs to the runner's user. `git init .` makes that directory a
repository, and git (>= 2.35.4) then refuses the next command (`git remote
add`) in a repository owned by another user. The old `git clone .` was never
followed by another git command, so never tripped it. The fetch-by-SHA test
above ran in a directory I owned, so could not show it.

Fix: `git config --global --add safe.directory "$PWD"` before `git init`, in
all three templates (as actions/checkout does; global config of the
throwaway container). On the forgejo host job (ci.yaml) the workspace is the
runner user's own, so a no-op -- kept for uniformity. Regenerated.

Reproduced locally in the CI image, root in a directory owned by uid 1029:

```bash
docker run --rm -v $D:/__w/ws -w /__w/ws -e GITHUB_SHA=$SHA docker-xo-builder:v2 sh -ec \
  'git init --quiet . && git remote add origin https://github.com/Rconybea/xo-umbrella2.git && ...'
# old: "detected dubious ownership"; with the safe.directory line first: checked out dcfe96bd...
```

Umbrella: not yet committed.
