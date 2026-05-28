# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A collection of [void-packages](https://github.com/void-linux/void-packages)-style `template` files for building Hyprland and its ecosystem on Void Linux. It is **not** a fork of void-packages — it overlays on top of a fresh clone of it at build time.

Packages are published as a binary XBPS repository for x86_64-glibc, x86_64-musl, aarch64-glibc, and aarch64-musl.

## Repository Structure

- `srcpkgs/<name>/template` — xbps-src build templates (the main content of this repo)
- `common/shlibs` — shared library → package mappings, appended to void-packages' `common/shlibs` at build time
- `scripts/` — CI build pipeline scripts (run inside the GitHub Actions container)

## Template Format

Each `srcpkgs/<name>/template` is a shell-sourced file following [void-packages template conventions](https://github.com/void-linux/void-packages/blob/master/Manual.md). Key fields:

- `pkgname`, `version`, `revision` — package identity
- `build_style` — e.g. `cmake`, `meson`
- `hostmakedepends` / `makedepends` — build-time dependencies
- `distfiles` + `checksum` — upstream source tarball and its SHA256
- musl-specific overrides go in `if [ "$XBPS_TARGET_LIBC" = "musl" ]; then … fi` blocks
- Sub-packages (e.g. `-devel`) are declared with `<pkgname>_package()` functions

## Updating a Package

1. Bump `version` and update `revision=1`
2. Download the new upstream tarball and compute its SHA256: `sha256sum <file>`
3. Update `checksum` with the new hash
4. If any patches were applied, verify they still apply cleanly
5. Update the corresponding entry in `common/shlibs` if a shared library soname changed

## Build Pipeline (CI / Manual)

The CI pipeline in `.github/workflows/build-latest.yml` runs these scripts in order:

1. `scripts/set-environment` — sets `RESULT_NAME` and `BUILD_ARGS` env vars based on target arch
2. `scripts/generate-build-order.kts` — Kotlin script that topologically sorts `srcpkgs/` by intra-repo dependencies, writing `/work/build-order`
3. `scripts/clone-and-prepare` — clones void-packages, merges `common/shlibs`, copies `srcpkgs/`, bootstraps xbps-src
4. `scripts/build-packages` — iterates `/work/build-order`, calling `xbps-src pkg <name>` for each
5. `scripts/index-packages` / `scripts/sign-packages` / `scripts/push-repository` — package the results into a signed XBPS binary repository branch

## Manually Building Locally

```sh
# One-time setup
git clone https://github.com/void-linux/void-packages
cd void-packages && ./xbps-src binary-bootstrap && cd ..

git clone https://github.com/Makrennel/hyprland-void
cd hyprland-void

cat common/shlibs >> ../void-packages/common/shlibs
cp -r --remove-destination srcpkgs/* ../void-packages/srcpkgs

# Build a single package
cd ../void-packages
./xbps-src pkg hyprland

# Cross-compile (musl example)
./xbps-src -a x86_64-musl pkg hyprland
```

## CI Checks (PRs)

`.github/workflows/check.yml` runs on PRs that touch `srcpkgs/`:
- **xlint** — runs `xtools` linters against changed templates
- **build** — builds changed packages across all supported arch/libc combinations

To skip CI on a PR add `[ci skip]` to the PR title or body.

## Contributing Guidelines

- Cross-compile test new packages with `./xbps-src -a aarch64-musl pkg <name>` before submitting
- Use your own name/email in `maintainer=` on new templates
- Commit changes separately (one package per commit when possible)
- Do not make unrelated cleanups in the same commit as a version bump
