# 03 -- the CI builder's toolchain as a nix-shell: `xo-builder-shell`

Status: open
Type: feature / tooling
Raised: 2026-10-11 (RC), for `.xo-backlog/xo-websock/issues/19`

The cmake CI runs in the `docker-xo-builder` image
(`pkgs/docker-xo-builder.nix`).  Reproducing a CI result locally -- or
producing output that must match CI's, like a clang golden file
(`xo-websock/issues/19`) -- needs that image's toolchain.  Expose the
image's contents as a nix-shell, so the same store paths are one command
away without docker:

```bash
nix-shell ci-nxfs.nix -A xo-builder-shell     # this host (nxfs, relocated store)
nix-shell ci.nix      -A xo-builder-shell     # vanilla <nixpkgs>
```

## Why the shell matches the image

Same derivations, so same store paths.  Checked 2026-10-11 -- the image's
compilers, as CI task 612 printed them, against `ci-nxfs.nix`'s nixpkgs:

```bash
cd "$(xo-sdlc --umbrella2)"
nix-instantiate --eval --strict --json -E 'let p = import ../nixpkgs { overlays = import ./overlays.nix; };
  in [ "${p.llvmPackages_18.clang}" "${p.gcc}" ]'
# ["/home/roland/nixroot/nix/store/6nhbii41ar7m138zqi2y6211nl0c421r-clang-wrapper-18.1.8",
#  "/home/roland/nixroot/nix/store/kf84dj7rkdca1yryr9j198s3sgyia44d-gcc-wrapper-14.3.0"]
ssh vpn1 'zstd -dc /var/lib/forgejo/actions_log/roland/xo-umbrella2/64/612.log.zst' | grep -m2 -E '^\S+ (c\+\+|clang\+\+) '
# identical paths
```

(The image passes `clang = llvmPackages1.clang`, `llvmPackages1 =
pkgs.llvmPackages_18` -- `xo.nix`.)  clang there compiles against gcc
14.3.0's libstdc++ (`clang++ -v`: `-isystem .../gcc-14.3.0/include/c++/14.3.0`).

## Plan

- Factor the image's `contents` list out of `pkgs/docker-xo-builder.nix`
  into one definition both use: the image as `contents`, a `mkShell` as
  `packages`.  The list is the single source of truth; the shell can't drift
  from the image.
- Expose the shell from `ci.nix` as `xo-builder-shell` (so `ci-nxfs.nix`,
  which imports `ci.nix`, has it too).
- Compiler choice as in CI: the workflow sets `CC` / `CXX` per matrix entry
  (`.forgejo/workflows/ci-cmake.yaml`), so inside the shell
  `CC=clang CXX=clang++` gives CI's clang build.  No second shell over
  `clangStdenv`.

**Differs from the image, deliberately:** the image merges every package
under `/include`, `/lib` and points the compiler there (`NIX_CFLAGS_COMPILE`,
`NIX_LDFLAGS`, `PKG_CONFIG_PATH` in its `config.Env`); the shell uses
nixpkgs' setup hooks, per-package paths.  Include FLAGS differ; the headers
and compilers are the same derivations.  Unverified that nothing observable
depends on the difference -- `xo-websock/issues/19`'s first clang golden,
compared with CI's own output, is the check.

**Done when:** `nix-shell ci-nxfs.nix -A xo-builder-shell --run 'clang++
--version; command -v clang++ g++ cmake'` names the store paths CI prints,
and the image still builds (`nix-build -A xo.docker-xo-builder`, per the
file's header; `default.nix` -> `shells.nix`) with an unchanged contents
list.
