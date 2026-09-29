# TileDB for GenomicsDB: high-performance germline calling

This is a fork of [datma-health/TileDB](https://github.com/datma-health/TileDB) (formerly OmicsDataAutomation/TileDB), the TileDB 0.x-derived storage engine that [GenomicsDB](https://github.com/GenomicsDB/GenomicsDB) builds as a submodule. It is not [TileDB-Inc/TileDB](https://github.com/TileDB-Inc/TileDB) (TileDB 2.x), which has a different API and on-disk format.

Its default branch, `high_performance_germline_calling`, is upstream `master` plus the few commits listed below, one per change. It is used by the branch of the same name in [tfenne/GenomicsDB](https://github.com/tfenne/GenomicsDB), which carries the GenomicsDB side of the joint-calling work in [tfenne/gatk](https://github.com/tfenne/gatk).

For TileDB itself (installation, tutorials, the C API), see the [upstream README](https://github.com/datma-health/TileDB/blob/master/README.md) and [wiki](https://github.com/datma-health/TileDB/wiki).

## Changes on this branch

Each entry is one commit on top of upstream `master`, oldest first. None of them change the on-disk format, and arrays written with this branch and with upstream are interchangeable.
