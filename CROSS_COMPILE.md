# Cross-Compilation Guide

This project uses [`cross`](https://github.com/cross-rs/cross) (v0.2.5) with custom Docker images to cross-compile for multiple targets. All configuration lives in three places:

| File | Purpose |
|------|---------|
| `Cross.toml` | Maps each target to its custom Dockerfile |
| `.cargo/config.toml` | Sets per-target `rustflags` (musl targets only) |
| `Dockerfile.cross-*` | Custom build containers for each target |

## Prerequisites

- [cross](https://github.com/cross-rs/cross) v0.2.5: `cargo install cross --version 0.2.5`
- Docker (BuildKit enabled)
- Rust toolchain specified in `rust-toolchain.toml`

## Supported Targets

### `x86_64-unknown-linux-gnu`

```bash
cross build --release --target x86_64-unknown-linux-gnu
```

- **Dockerfile:** `Dockerfile.cross-gnu`
- **Base image:** Ubuntu 20.04 (the stock cross image uses Ubuntu 16.04 which is too old)
- **Linker:** System `gcc` (native compilation inside the container)
- **OpenSSL:** System `libssl-dev` (dynamically linked)
- **Status:** ✅ Working

**Why a custom image?** The default `cross` image for this target ships Ubuntu 16.04 with glibc 2.23 and libclang 3.8. The `bindgen` crate (used by `librocksdb-sys`) requires libclang ≥ 5.0, and recent Rust crate build scripts require glibc ≥ 2.28.

---

### `aarch64-unknown-linux-gnu`

```bash
cross build --release --target aarch64-unknown-linux-gnu
```

- **Dockerfile:** `Dockerfile.cross-gnu-aarch64`
- **Base image:** Ubuntu 20.04 with cross-compiler copied from the stock cross image
- **Linker:** `aarch64-linux-gnu-gcc`
- **OpenSSL:** Vendored (compiled from source via `openssl` crate `vendored` feature)
- **Status:** 🔧 In progress (same glibc/libclang issues as x86_64-gnu, plus cross-compilation of OpenSSL C library)

**Why a custom image?** Same as x86_64-gnu (old Ubuntu 16.04 base), plus the aarch64 cross-compiler toolchain must be copied from the stock cross image. The `BINDGEN_EXTRA_CLANG_ARGS` environment variable is set to `--sysroot=/usr/aarch64-linux-gnu` so that `bindgen`/`libclang` can find the correct cross-compilation headers.

**Key environment variables set in the Dockerfile:**

| Variable | Value | Purpose |
|----------|-------|---------|
| `CC_aarch64_unknown_linux_gnu` | `aarch64-linux-gnu-gcc` | C compiler for target |
| `CXX_aarch64_unknown_linux_gnu` | `aarch64-linux-gnu-g++` | C++ compiler for target |
| `AR_aarch64_unknown_linux_gnu` | `aarch64-linux-gnu-ar` | Archiver for target |
| `BINDGEN_EXTRA_CLANG_ARGS_aarch64_unknown_linux_gnu` | `--sysroot=/usr/aarch64-linux-gnu` | Headers for bindgen |
| `PKG_CONFIG_ALLOW_CROSS_aarch64_unknown_linux_gnu` | `1` | Allow pkg-config in cross builds |

---

### `x86_64-unknown-linux-musl`

```bash
cross build --release --target x86_64-unknown-linux-musl
```

- **Dockerfile:** `Dockerfile.cross-musl`
- **Base image:** Stock `cross` image (`ghcr.io/cross-rs/x86_64-unknown-linux-musl:0.2.5`, Ubuntu 20.04)
- **Linker:** `musl-gcc-wrapper` (custom wrapper around `x86_64-linux-musl-gcc`)
- **OpenSSL:** Vendored (compiled from source)
- **Output:** Dynamically linked against musl libc (`ld-musl-x86_64.so.1`)
- **Status:** ✅ Working

**Why a custom image?** Two issues required workarounds:

1. **OpenSSL not available for musl.** The `openssl` crate is added to `Cargo.toml` with `features = ["vendored"]` to compile OpenSSL from source rather than depending on system headers.

2. **Musl linker symbol ordering.** When statically linking Rust's `.rlib` archives, symbols like `pthread_rwlock_rdlock`, `dladdr`, and `pow` (which all live in musl's `libc`) are referenced by `libunwind.a`, `libopenssl_sys`, and `librocksdb_sys` but appear *after* `-lc` in the link command. A linker wrapper script appends `-Wl,--no-as-needed -lpthread -ldl -lm -lc` at the end of the link command to resolve these.

**Why `-C target-feature=-crt-static`?** Set in `.cargo/config.toml`. By default, Rust's musl target uses fully static linking (`crt-static`), but this causes unresolvable symbol ordering issues with the vendored OpenSSL and RocksDB C/C++ libraries. Disabling it produces a dynamically linked musl binary (linked against `ld-musl-x86_64.so.1`), which resolves the link ordering problems while still using musl libc.

---

### `aarch64-unknown-linux-musl`

```bash
cross build --release --target aarch64-unknown-linux-musl
```

- **Dockerfile:** `Dockerfile.cross-musl-aarch64`
- **Base image:** Ubuntu 20.04 with musl cross-toolchain copied from stock cross image
- **Linker:** `musl-gcc-wrapper` (custom wrapper around `aarch64-linux-musl-gcc`)
- **OpenSSL:** Vendored (compiled from source)
- **Output:** Dynamically linked against musl libc (`ld-musl-aarch64.so.1`)
- **Status:** ✅ Working

**Why a custom image?** Same musl issues as x86_64-musl (vendored OpenSSL, linker wrapper), plus:

- The stock cross image uses Ubuntu 18.04 with glibc 2.27, which is too old for host-side build scripts that require glibc ≥ 2.28. A multi-stage Dockerfile copies the musl cross-toolchain into an Ubuntu 20.04 base.
- `libclang-dev` must be installed for `bindgen` (used by `librocksdb-sys`).
- `BINDGEN_EXTRA_CLANG_ARGS` is set to point at the musl sysroot and GCC includes so `bindgen` can find `stddef.h` and other standard headers.

**Key environment variables set in the Dockerfile:**

| Variable | Value | Purpose |
|----------|-------|---------|
| `CC_aarch64_unknown_linux_musl` | `aarch64-linux-musl-gcc` | C compiler for target |
| `CXX_aarch64_unknown_linux_musl` | `aarch64-linux-musl-g++` | C++ compiler for target |
| `AR_aarch64_unknown_linux_musl` | `aarch64-linux-musl-ar` | Archiver for target |
| `BINDGEN_EXTRA_CLANG_ARGS_aarch64_unknown_linux_musl` | `--sysroot=... -I...` | Headers for bindgen |

---

### `x86_64-pc-windows-gnu`

```bash
cross build --release --target x86_64-pc-windows-gnu
```

- **Dockerfile:** `Dockerfile.cross-windows-gnu`
- **Base image:** Ubuntu 22.04 with MinGW-w64 (posix threading model)
- **Linker:** `mingw-gcc-wrapper` (custom wrapper around `x86_64-w64-mingw32-gcc`)
- **OpenSSL:** Vendored (compiled from source with `no-quic`)
- **Output:** Windows PE64 executable (`.exe`), 67 MB
- **Status:** ✅ Working

**Why a custom image?** The stock cross image uses Ubuntu 18.04 with GCC 7.3 and glibc 2.27, all too old. A multi-stage Dockerfile uses Ubuntu 22.04 with MinGW-w64 GCC 10.

**Key workarounds:**

1. **MinGW QUIC headers missing.** OpenSSL 3.5.5 enables QUIC by default, which uses `SIO_UDP_NETRESET` — an API not in MinGW headers. Setting `OPENSSL_CONFIGURE_ARGS="no-quic"` disables it.

2. **MinGW threading model.** RocksDB uses `std::mutex` which requires the POSIX threading model. The default MinGW GCC uses Win32 threads. Fixed with `update-alternatives --set x86_64-w64-mingw32-gcc /usr/bin/x86_64-w64-mingw32-gcc-posix`.

3. **Pthread link ordering.** RocksDB, `libstdc++`, and `libgcc_eh` all reference `pthread_*` symbols. Rust passes `-nodefaultlibs` to the GCC driver, which prevents default library resolution. The linker wrapper script injects the full path to `libpthread.a` (plus `-lkernel32 -lmsvcrt` for winpthread's own dependencies) just before `-nodefaultlibs` to resolve all pthread references.

**Key environment variables set in the Dockerfile:**

| Variable | Value | Purpose |
|----------|-------|---------|
| `CC_x86_64_pc_windows_gnu` | `x86_64-w64-mingw32-gcc` | C compiler for target |
| `CXX_x86_64_pc_windows_gnu` | `x86_64-w64-mingw32-g++` | C++ compiler for target |
| `AR_x86_64_pc_windows_gnu` | `x86_64-w64-mingw32-ar` | Archiver for target |
| `BINDGEN_EXTRA_CLANG_ARGS_x86_64_pc_windows_gnu` | `--sysroot=... -I...` | Headers for bindgen |
| `OPENSSL_CONFIGURE_ARGS` | `no-quic` | Disables QUIC in vendored OpenSSL |

---

## Common Build Dependencies

All custom Dockerfiles install these packages:

| Package | Required by |
|---------|-------------|
| `perl`, `make` | Vendored OpenSSL build system (Configure + make) |
| `protobuf-compiler`, `libprotobuf-dev` | `drasi-reaction-grpc` and `drasi-source-grpc` (prost/tonic proto compilation) |
| `libclang-dev` | `bindgen` crate (used by `librocksdb-sys` for FFI bindings) |
| `libssl-dev` | OpenSSL headers (GNU targets only; musl targets use vendored OpenSSL) |

## File Reference

```
Cross.toml                      # Target → Dockerfile mapping
.cargo/config.toml              # rustflags for musl targets (-crt-static) and windows (+crt-static)
Cargo.toml                      # openssl = { version = "0.10", features = ["vendored"] }
Dockerfile.cross-musl           # x86_64-unknown-linux-musl container
Dockerfile.cross-musl-aarch64   # aarch64-unknown-linux-musl container
Dockerfile.cross-gnu            # x86_64-unknown-linux-gnu container
Dockerfile.cross-gnu-aarch64    # aarch64-unknown-linux-gnu container
Dockerfile.cross-windows-gnu    # x86_64-pc-windows-gnu container
```

## Troubleshooting

### `Could not find openssl` / `openssl.pc not found`
The `openssl` crate with `features = ["vendored"]` in `Cargo.toml` handles this by building OpenSSL from source. Ensure `perl` and `make` are installed in the Docker image.

### `Could not find protoc`
Install `protobuf-compiler` and `libprotobuf-dev` in the Docker image.

### `Unable to find libclang` / `libclang function not supported`
Install `libclang-dev` (≥ 5.0) in the Docker image. The stock cross images ship libclang 3.8 which is too old for recent versions of `bindgen`.

### `GLIBC_2.28 not found` / `GLIBC_2.29 not found`
The stock cross image base OS is too old. Use a custom Dockerfile with Ubuntu 20.04+ as the base.

### `undefined reference to pthread_rwlock_rdlock` / `dladdr` / `pow` (musl targets)
This is a linker symbol ordering issue specific to musl static linking. The linker wrapper script (`musl-gcc-wrapper`) in the Dockerfile resolves this by appending `-Wl,--no-as-needed -lpthread -ldl -lm -lc` at the end of the link command.

### `stddef.h file not found` (aarch64 targets)
Set `BINDGEN_EXTRA_CLANG_ARGS` to include the cross-compilation sysroot and GCC internal include paths. See the Dockerfile for the exact values.

### `cc1: execvp: No such file or directory` (aarch64-gnu)
The cross-compiler's GCC support files (`cc1`, `cc1plus`) are not in `PATH`. Ensure `/usr/local/` from the cross image is fully copied, or install the `gcc-aarch64-linux-gnu` package.

### `SIO_UDP_NETRESET` / QUIC errors (Windows target)
OpenSSL 3.5.5 enables QUIC by default, which references Windows API constants not in MinGW headers. Set `OPENSSL_CONFIGURE_ARGS="no-quic"` in the Dockerfile to disable QUIC.

### `std::mutex` / `mutex not a member of std` (Windows target)
MinGW-w64 defaults to the Win32 threading model, which doesn't support `std::mutex`. Switch to the POSIX threading model with `update-alternatives --set x86_64-w64-mingw32-gcc /usr/bin/x86_64-w64-mingw32-gcc-posix`.

### `undefined reference to pthread_mutex_lock` (Windows target)
Rust passes `-nodefaultlibs` which prevents the GCC driver from resolving pthread. The linker wrapper in `Dockerfile.cross-windows-gnu` injects the full path to `libpthread.a` (plus `-lkernel32 -lmsvcrt`) before `-nodefaultlibs` to resolve all pthread references.
