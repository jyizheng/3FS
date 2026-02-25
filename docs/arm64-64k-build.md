# Building 3FS on ARM64 with 64KB Page Kernel

This guide covers building 3FS on aarch64 machines with 64KB page size kernels (e.g., NVIDIA GH200/NVL72).

## Prerequisites

- ARM64 machine (cannot cross-compile from x86)
- Docker with `open3fs/3fs-build:20250617-openeuler2203` image
- Source cloned with submodules: `git clone --recurse-submodules https://github.com/jyizheng/3FS.git`

## Build Environment

| Component | Details |
|-----------|---------|
| Build image | `open3fs/3fs-build:20250617-openeuler2203` (openEuler 22.03) |
| Compiler | BiSheng Compiler 4.2.0 (clang-14 compatible) |
| Rust | cargo 1.87.0 / rustc (pre-installed at `/root/.cargo/bin/`) |
| jemalloc | Git submodule at `third_party/jemalloc` (built from source) |

## What Changed for ARM64 64KB Page Support

### 1. Patches (patches/apply.sh)

These patch submodules and source files that cannot be modified directly in this repo:

| Patch | Target | Issue |
|-------|--------|-------|
| `folly.patch` | `third_party/folly/` | **StampedPtr sign-extension bug** -- `unpackPtr()` uses arithmetic right shift, corrupting user-space pointers on 64KB page kernels where bit 47=1. Causes SIGSEGV in all RDMA components. |
| `rocksdb.patch` | `third_party/rocksdb/` | Cassandra TTL namespace conflict on ARM64 |
| `rocksdb.patch.2` | `third_party/rocksdb/` | CMakeLists: suppress unused-but-set-variable warning, disable db/c.cc |
| `ibconnect-pkey-index.patch` | `src/common/net/ib/IBConnect.cc` | Server uses own configured pkey_index on accept instead of client-provided value. Required when nodes have different P_Key table layouts. |

### 2. Build System Changes (committed directly)

**cmake/Jemalloc.cmake -- `--with-lg-page=16`**

jemalloc is a git submodule (third_party/jemalloc), built from source during cmake. The `--with-lg-page=16` flag tells jemalloc the page size is 2^16 = 64KB. Without this, jemalloc assumes 4KB pages and causes memory corruption.

**cmake/ApacheArrow.cmake -- `-DARROW_JEMALLOC=OFF -DARROW_MIMALLOC=OFF`**

Apache Arrow is fetched via ExternalProject_Add (git clone during build). Arrow ships its own bundled jemalloc which does not support `--with-lg-page`. Disabling it forces Arrow to use system malloc instead.

**CMakeLists.txt -- Relaxed compiler warnings**

BiSheng Compiler (clang) on aarch64 produces additional nullability and GNU extension warnings. Changed `-Werror` to allow these without failing the build.

### 3. Cargo / Rust

3FS includes Rust crates built via cargo during cmake:

- `src/storage/chunk_engine` -- Storage chunk engine (core I/O)
- `src/client/trash_cleaner` -- Trash cleanup utility
- `src/lib/rs/hf3fs-usrbio-sys` -- UsrbIo FFI bindings

cmake/AddCrate.cmake invokes `cargo build --release` and links the static libraries into CMake targets. Cargo and rustc are pre-installed in the build image at `/root/.cargo/bin/` -- no installation during build. This is also why cross-compilation is not possible: cargo compiles native aarch64 code.

## Quick Build

### Option A: Docker build from source directory

    cd /path/to/3FS

    # Apply patches (idempotent -- skips already-applied patches)
    cd patches && bash apply.sh && cd ..

    # Build inside container
    docker run --rm \
      -v $(pwd):/3fs -w /3fs \
      open3fs/3fs-build:20250617-openeuler2203 \
      bash -c '
        export PATH="/root/.cargo/bin:$PATH"
        ln -sf /opt/bisheng/BiShengCompiler-4.2.0-aarch64-linux/bin/clang-format /usr/bin/clang-format-14
        git config --global --add safe.directory /3fs
        git tag -f 260224 HEAD
        mkdir -p build && cd build
        cmake .. \
          -DCMAKE_BUILD_TYPE=RelWithDebInfo \
          -DSHUFFLE_METHOD=stdshuffle \
          -DENABLE_FUSE_APPLICATION=ON \
          -DBUILD_TESTS=OFF \
          -DCMAKE_C_COMPILER=/opt/bisheng/BiShengCompiler-4.2.0-aarch64-linux/bin/clang \
          -DCMAKE_CXX_COMPILER=/opt/bisheng/BiShengCompiler-4.2.0-aarch64-linux/bin/clang++
        make -j$(nproc)
      '

### Option B: Dockerfile (self-contained)

See Dockerfile.dev-github -- clones from GitHub, applies patches, builds everything:

    docker build -f Dockerfile.dev-github -t 3fs-dev:arm64-64k .

### Incremental Rebuild

After modifying source, only recompile changed files:

    docker run --rm -v $(pwd):/3fs -w /3fs/build \
      open3fs/3fs-build:20250617-openeuler2203 \
      bash -c 'export PATH="/root/.cargo/bin:$PATH" && make -j$(nproc)'

## Build Artifacts

| Binary | Purpose |
|--------|---------|
| `build/bin/mgmtd_main` | Management daemon |
| `build/bin/meta_main` | Metadata server |
| `build/bin/storage_main` | Storage server |
| `build/bin/hf3fs_fuse_main` | FUSE client |
| `build/bin/admin_cli` | CLI admin tool |
| `build/bin/monitor_collector_main` | Metrics collector |
| `build/src/lib/api/libhf3fs_api_shared.so` | UsrbIo shared library |

## CMake Options

| Option | Value | Notes |
|--------|-------|-------|
| `CMAKE_BUILD_TYPE` | `RelWithDebInfo` | Release with debug symbols |
| `SHUFFLE_METHOD` | `stdshuffle` | Required for clang (GCC-specific g++11 method incompatible) |
| `ENABLE_FUSE_APPLICATION` | `ON` | Build FUSE client |
| `BUILD_TESTS` | `OFF` | Skip tests (faster build) |

## Key Notes

- **git tag must be numeric** (e.g., 260224). VersionInfo.cc uses BUILD_TAG as uint32_t. Semver tags like v0.1.0 cause build failure.
- **clang-format-14 symlink** needed: the build image only has BiSheng's clang-format, not clang-format-14.
- **Full build time**: ~20-40 minutes. Incremental: ~2-5 minutes.
- All binaries are ELF 64-bit LSB pie executable, ARM aarch64, dynamically linked.
