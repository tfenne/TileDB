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
