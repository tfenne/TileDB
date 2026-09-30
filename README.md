# TileDB for GenomicsDB: high-performance germline calling

This is a fork of [datma-health/TileDB](https://github.com/datma-health/TileDB) (formerly OmicsDataAutomation/TileDB), the TileDB 0.x-derived storage engine that [GenomicsDB](https://github.com/GenomicsDB/GenomicsDB) builds as a submodule. It is not [TileDB-Inc/TileDB](https://github.com/TileDB-Inc/TileDB) (TileDB 2.x), which has a different API and on-disk format.

Its default branch, `high_performance_germline_calling`, is upstream `master` plus the few commits listed below, one per change. It is used by the branch of the same name in [tfenne/GenomicsDB](https://github.com/tfenne/GenomicsDB), which carries the GenomicsDB side of the joint-calling work in [tfenne/gatk](https://github.com/tfenne/gatk).

For TileDB itself (installation, tutorials, the C API), see the [upstream README](https://github.com/datma-health/TileDB/blob/master/README.md) and [wiki](https://github.com/datma-health/TileDB/wiki).

## Changes on this branch

Each entry is one commit on top of upstream `master`, oldest first. None of them change the on-disk format, and arrays written with this branch and with upstream are interchangeable.

### Faster tile reads

Three changes to how sparse reads fetch tiles, which together cut GATK GnarlyGenotyper's runtime over a 1,000-sample GenomicsDB workspace by about a third. Query results are unchanged.

- **Attribute files stay open across tile reads.** With the default mmap read method, every tile read opened and closed its attribute file, and `open()` was about 15% of a GnarlyGenotyper profile. A `ReadState` now keeps each attribute file (and variable-sized attribute file) open until it is destroyed, within a process-wide budget: `TILEDB_MAX_CACHED_READ_FILE_HANDLES` if set, otherwise half the soft `RLIMIT_NOFILE`. Past the budget, files are opened per tile as before.
- **Compressed tiles are read with `pread` instead of `mmap`.** They are always decompressed into a separate buffer, so mapping them saved no copy but cost an `mmap`/`munmap` pair and fresh page faults per tile. They are now read from the held file descriptor into a reused buffer. Uncompressed tiles are still mapped.
- **C API sanity checks are inline.** `sanity_check()` runs per attribute per cell through `tiledb_array_iterator_get_value()`; building its error message in the same function made every check an out-of-line call with a large stack frame. Error reporting now lives in a separate `noinline` function.

Measured one after another on GnarlyGenotyper, 1,000 samples, chr20:60,001-16,000,000, LZ4-compressed workspace: held file descriptors 1,222 → 911 s (−25%), then `pread` 1,001 → 889 s (−11%, paired runs), then the inline checks, together with a GenomicsDB change, 859 → 828 s (−3.6%).

### zstd compiled in

- **zstd is built into TileDB instead of loaded at run time.** Upstream loads `libzstd.so.1` with `dlopen` the first time a zstd codec is created, and throws if it isn't found; through GenomicsDB's JNI layer that surfaced as an abort of the JVM on machines without libzstd. TileDB now compiles zstd 1.5.7 (downloaded at configure time, checked against its SHA-256) into its own objects, the way it already builds LZ4. zstd's symbols are hidden, so they neither clash with nor bind to another zstd in the same process.
- **The decompression context is freed with `ZSTD_freeDCtx`.** Each thread's decompression context was freed with `ZSTD_freeCCtx` when the thread exited. The codec now uses zstd's typed contexts, so the mismatch no longer compiles.

Tiles are compressed by zstd 1.5.7 whatever the system has installed. Another zstd version can compress the same tile to different bytes, and each reads the other's output.

### Builds with current toolchains

TileDB and the SDKs it builds (AWS SDK 1.8, google-cloud-cpp 1.24, azure-storage-cpplite) build without local patches or environment settings with CMake 4, GCC 15 and clang 21 on AlmaLinux 8 (glibc 2.28, x86-64 and aarch64), and Apple clang 21 targeting macOS 14 on arm64. The SDK versions are unchanged.

- **CMake 4.** TileDB requires CMake 3.22. The SDK builds are given `CMAKE_POLICY_VERSION_MINIMUM=3.5`, since their own `cmake_minimum_required()` predates what CMake 4 accepts.
- **One toolchain.** The SDKs are built with TileDB's compilers and macOS deployment target, and google-cloud-cpp with TileDB's OpenSSL.
- **Compiler fixes.** The AWS SDK gets the standard includes it relied on older compilers to pull in; google-cloud-cpp's warnings are no longer errors; abseil's `-Xarch` flags are fixed (from upstream `develop`); azure-storage-cpplite's `NULL`-as-`std::string` warning, in code TileDB never calls, is no longer an error.
- **OpenSSL 3 digests in google-cloud-cpp.** The OpenSSL 3 code paths in the google-cloud-cpp patch sized MD5 and SHA-256 buffers for SHA-1, freed the SHA-256 digest before returning it, and cut MD5 digests at the first zero byte. They now use fixed-size buffers and explicit lengths.
- **`char` is signed everywhere.** On Linux aarch64 `char` is unsigned, so `TILEDB_EMPTY_CHAR` (`CHAR_MAX`) was 255 there and 127 elsewhere: missing values in arrays written on x86-64 or macOS read back there as 127, and missing values written on aarch64 read back elsewhere as −1. TileDB is now built with `-fsigned-char`, as must be anything that includes its headers.
- **Reproducible downloads.** Every SDK download is pinned and checked against its SHA-256. The AWS SDK is 1.8.187, the release the previously downloaded `1.8.x` branch points to.
- **No builds in `$HOME`.** The SDKs are built in `TILEDB_DEPS_INSTALL_DIR`, the build tree by default, unless `AWSSDK_ROOT_DIR`/`GCSSDK_ROOT_DIR` point at existing installs, instead of `$HOME/awssdk-install` and `$HOME/gcssdk-install`. A parent project can point `TILEDB_DEPS_INSTALL_DIR` elsewhere to share the SDK builds between build trees.
- **Subproject-safe configure.** Configuring TileDB updates only its own submodules; as a subproject it used to reset every submodule of the parent project. On macOS the libuuid lookup no longer picks up `Kernel.framework`, whose headers shadowed the SDK's C headers. `TILEDB_DISABLE_TESTS` no longer breaks configuring the examples.

### Cheaper iterator steps

- **The array iterator no longer copies its attribute list on every step.** `ArrayIterator::next()` runs once per cell, and it copied the array's attribute-id vector each time: an allocation and a free per cell. It now takes a reference.

Measured together with three GenomicsDB changes to its per-cell and per-record work, on GnarlyGenotyper, 1,000 samples, chr20:1-16,000,000: 475.0 → 428.2 s (−9.9%).
