# Patch notice — gmssl-rs 0.1.1 + gmssl-rs-sys 0.1.0

This repository is a **patched copy of the published crates.io sources**
([`gmssl-rs`](https://crates.io/crates/gmssl-rs) 0.1.1 and
[`gmssl-rs-sys`](https://crates.io/crates/gmssl-rs-sys) 0.1.0), laid out the way upstream's own
repository lays them out (`gmssl/` + `gmssl-sys/`, the former depending on the latter via
`path`). It exists so a downstream crate can get **working Windows/MSVC builds** with a single
`[patch.crates-io]` entry while upstream's fixes are unreleased.

| Commit | Content |
| --- | --- |
| 1 | crates.io sources of both crates, reassembled into the upstream workspace layout (no functional change) |
| 2 | **the only functional change**: the two Windows fixes, one file each — `git show` is the complete diff |
| 3 | this document, `LICENSE`, `.gitignore`, README notice |

## Why this exists

Two independent defects make the *published* pair unusable on Windows MSVC:

### 1. `gmssl-sys/build.rs` (in `gmssl-sys/`)

| Failure | Cause | Fix |
| --- | --- | --- |
| `C1083: Cannot open include file: 'dlfcn.h'` / `'netdb.h'` | GmSSL's headers gate their Windows code on `#ifdef WIN32`, but MSVC only predefines `_WIN32`. CMake's `Windows-MSVC` platform module normally injects `/DWIN32`, yet the `cmake` crate overwrites `CMAKE_C_FLAGS` / `CMAKE_C_FLAGS_RELEASE` and drops it, so those translation units take the POSIX path. | `cmake_cfg.cflag("-DWIN32")` |
| The build installs into `C:/Program Files/GmSSL`, so no `lib/` shows up under `OUT_DIR` | GmSSL's `CMakeLists.txt` hardcodes `set(CMAKE_INSTALL_PREFIX "C:/Program Files/GmSSL")`, which overrides the value passed via `-D`. | patch the file for the duration of the build, restore it afterwards |
| `CMake build completed but no lib/ directory found` | The Visual Studio generator is multi-config: libraries land in `lib/Release/`. | `find_lib_dir()` |
| Unresolved externals for CryptoAPI (`CryptAcquireContext`, …) | GmSSL's `rand_win.c` uses the Windows CryptoAPI. | link `advapi32` |

### 2. `gmssl/src/pem_helpers.rs` (in `gmssl/`)

```text
error[E0425]: cannot find function `fmemopen` in crate `libc`
error[E0425]: cannot find function `open_memstream` in crate `libc`
```

The POSIX-only `fmemopen` / `open_memstream` were used to hand GmSSL's C APIs a `FILE*`;
upstream replaced them with `tmpfile()`-based variants behind `#[cfg(windows)]`.

**All of the above is already fixed on upstream `main`** — these commits are backports, taken
verbatim from `GmSSL/gmssl-rs@main` and verified byte-for-byte for `pem_helpers.rs`. What
upstream has *not* done is release it: `gmssl-rs-sys` is still 0.1.0 and `gmssl-rs` still 0.1.1
on crates.io, both published before the fixes landed.

## What is *not* changed

- **GmSSL stays at 3.1.1** — the source tree shipped inside the published `gmssl-rs-sys` crate
  (`gmssl-sys/GmSSL/`) is included as a normal directory, so builds need **no download and no
  submodule**.
- Package names, versions (`0.1.1` / `0.1.0`) and the whole public API are untouched.
  The `gmssl/Cargo.toml` and `gmssl-sys/Cargo.toml` here are the authors' original manifests
  (`Cargo.toml.orig` from the published crates); only cargo's packaged artifacts
  (`Cargo.lock`, `Cargo.toml.orig`, the normalized `Cargo.toml`) are not reproduced.

Upstream `main`, by contrast, now downloads a **GmSSL v3.2.0** tarball at build time and its
bindings target 3.2.0. Depending on `main` therefore bumps the C library (and its measured
performance) together with the bindings — this repository deliberately avoids that, because its
only goal is Windows support for the bytes that are already proven on macOS/Linux.

## Usage

```toml
[patch.crates-io]
gmssl-rs = { git = "https://github.com/jeromexiong/gmssl-rs-patched", rev = "<commit-sha>" }
```

One entry is enough: `gmssl-rs-sys` is a `path` dependency inside this repository, so it is
resolved from the same commit. Pin a `rev` (or `tag`) so builds stay reproducible, then
`cargo update -p gmssl-rs -p gmssl-rs-sys`.

⚠️ One downstream note: `gmssl-rs` 0.1.1 declares FFI for two symbols that GmSSL **3.2.0** has
but 3.1.1 does not (`x509_key_cleanup`, `zuc256_generate_keystream`). On macOS/Linux the static
archive is only pulled member-by-member, so it links; on MSVC those codegen units do get pulled
and you get `LNK2019` → `LNK1120`. Downstreams that do not expose X509/ZUC (e.g. only
SM2/SM3/SM4) can pass `/FORCE:UNRESOLVED` — and should then prove the DLL actually loads by
running tests against the built artifact. (Fixed here for a specific consumer via a `build.rs`,
not in this repository.)

## Removal

Delete the `[patch.crates-io]` entry as soon as upstream publishes a release containing these
fixes. Repositories `gmssl-rs-sys-patched` (sys-only, commit 1/2 of the same content) is
superseded by this one and can be deleted then too.

## Upstream nits worth reporting (not fixed here)

1. `gmssl-sys/Cargo.toml` (as published 0.1.0) points `repository` / `homepage` at
   `https://github.com/guanzhi/gmssl-rs`, which **404s** — the real project is
   [`GmSSL/gmssl-rs`](https://github.com/GmSSL/gmssl-rs).
2. Upstream `main` is self-contradictory about its C library: `.gitmodules` still pins the
   submodule at `d655c06b` (GmSSL **v3.1.1**) while `GMSSL_RELEASE_TAG` is `v3.2.0`.
   Since `locate_gmssl_source()` no longer looks at the submodule at all (it downloads the
   tarball unless `GMSSL_SOURCE_DIR` is set), that stale gitlink only misleads readers.
3. No release since 2026-05-31 ⇒ downstreams cannot get the Windows fixes without pinning git.

→ All three are addressed by [`GmSSL/gmssl-rs#3`](https://github.com/GmSSL/gmssl-rs/pull/3)
(filed 2026-10-09: GMSSL_DIR platform link libs fix, submodule removal, 0.2.0 release prep),
pending merge and `cargo publish`. Nit 1's manifest fix already landed on upstream `main`.

## Provenance and license

- Base bytes: crates.io `gmssl-rs` 0.1.1 + `gmssl-rs-sys` 0.1.0 (Apache-2.0), commit 1.
- Patches: `GmSSL/gmssl-rs@main` (Apache-2.0) — `gmssl-sys/build.rs`, `gmssl/src/pem_helpers.rs`, commit 2.
- Bundled `gmssl-sys/GmSSL/`: GmSSL 3.1.1, Apache-2.0 (`gmssl-sys/GmSSL/LICENSE`).
- This repository: Apache-2.0 (see `LICENSE`).
