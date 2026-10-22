# Upstream PR measurements

> Speed-up over stock TSan, **>1 = faster**. State of 22 Oct 2026.

| Column | What it is | Upstream PR |
|---|---|---|
| **EA** | Escape analysis: capture-tracking limit + escape through local slots | [llvm#192945](https://github.com/llvm/llvm-project/pull/192945) + [llvm#169896](https://github.com/llvm/llvm-project/pull/169896) |
| **DE** | Dominance-based elimination, reads and writes | [llvm#169897](https://github.com/llvm/llvm-project/pull/169897) |
| **DE, reads only** | DE that keeps every write checked (fix for the write-after-report gap) | [llvm#169897](https://github.com/llvm/llvm-project/pull/169897) with F1 |
| **DE + EA** | DE + capture limit + local slots | all three |

🟢 faster  ·  ⚪ no change  ·  🟡 unresolved (noise)  ·  🔴 slower  ·  🟰 identical code (EA: to stock; DE + EA: to DE), not timed  ·  ⏳ planned or running  ·  — not measured

## Machine

All timings: **focs**, Intel Xeon w9-3495X (56 cores, 112 threads, max 4.8 GHz), 250 GB RAM, Linux 6.8.
Each run is pinned to one half of the machine (CPUs 4-27, 60-83; 8-thread runs use 4-11, 1-thread runs CPU 4); a
run is discarded if the other half is busy. Every arm is timed in 2-4 code layouts and the results pooled.
Removed-check counts (screening) were taken on apollo (AMD EPYC 9115) and are not timings.

## Results

| Program | Threads | EA | DE | DE, reads only | DE + EA |
|---|:-:|:-:|:-:|:-:|:-:|
| **Applications, -O0** | | | | | |
| SQLite | 8 | ⚪ 1.08¹ | ⏳ | ⏳ | ⏳ |
| FFmpeg | 8 | 🟢 **1.004** | ⏳ | ⏳ | ⏳ |
| memcached | 8 | ⚪ 0.99 | ⏳ | ⏳ | ⏳ |
| Redis | 8 | ⚪ 0.996 | ⏳ | ⏳ | ⏳ |
| MySQL | 8 | ⚪ 0.999 | ⏳ | ⏳ | ⏳ |
| **Applications, -O2** | | | | | |
| SQLite | 8 | 🟰 1.000 | ⏳ | ⏳ | ⏳ |
| FFmpeg | 8 | ⚪ 0.993 | ⏳ | ⏳ | ⏳ |
| memcached | 8 | 🟰 1.000 | ⏳ | ⏳ | ⏳ |
| Redis | 8 | 🟰 1.000 | ⚪ 0.997 | ⚪ 0.997 | 🟰 0.997 |
| MySQL | 8 | ⚪ 0.997 | ⏳ | ⏳ | ⏳ |
| **Benchmark kernels** (NPB, Rodinia, SPLASH-2x, PARSEC: small programs that model one algorithm, not applications) | | | | | |
| Rodinia hotspot | 8 | 🟰 1.000 | 🟢 **3.05** | 🟢 **3.21** | 🟰 3.05 |
| | 1 | 🟰 1.000 | 🟢 **1.10** | 🟢 **1.10** | 🟰 1.10 |
| NPB LU, class A | 8 | ⏳ | 🟢 **2.27** | 🟢 **2.24** | ⏳ |
| NPB LU, class W | 8 | ⏳ | 🟢 **2.02** | 🟢 **2.04** | ⏳ |
| NPB BT, class A | 8 | ⏳ | 🟢 **1.76** | 🟢 **1.47** | ⏳ |
| SPLASH-2x ocean_ncp | 8 | 🟰 1.000 | 🟢 **1.06** | 🟢 **1.07** | 🟰 1.06 |
| SPLASH-2x water_nsquared | 8 | 🟰 1.000 | 🟡 1.09 | 🟡 1.07 | 🟰 1.09 |
| | 1 | 🟰 1.000 | 🟢 **1.26** | 🟢 **1.26** | 🟰 1.26 |
| SPLASH-2x water_spatial | 8 | 🟰 1.000 | 🟡 1.10 | 🟡 1.08 | 🟰 1.10 |
| | 1 | 🟰 1.000 | 🟢 **1.26** | 🟢 **1.26** | 🟰 1.26 |
| Rodinia srad | 8 | 🟰 1.000 | ⚪ 1.02 | ⚪ 1.00 | 🟰 1.02 |
| | 1 | 🟰 1.000 | 🟢 **1.07** | 🟢 **1.04** | 🟰 1.07 |
| PARSEC streamcluster | 8 | 🟰 1.000 | ⚪ 0.98 | ⚪ 0.98 | 🟰 0.98 |
| PARSEC fluidanimate | 8 | 🟰 1.000 | ⚪ 0.99 | ⚪ 1.00 | 🟰 0.99 |
| Rodinia lavaMD | 8 | 🟰 1.000 | 🟡 ~1.1 | 🟡 ~1.1 | 🟰 ~1.1 |

¹ SQLite -O0 as a whole shows no change, but two tests do: checkpoint_starvation_1 🟢 **1.77×**, walthread1 🟢 **1.07×**.

In every column, EA means the two EA PRs (capture limit + local slots); the paper's broader escape analysis is not used.

Not timed: 254 programs were screened by the share of checks DE removes; the rest fell below ~3 % or did not
separate from noise (NPB IS/SP/FT/CG/MG/EP, GAP, PARSEC/SPLASH-2x others, pbbs, LULESH, HPCCG, CoMD, miniFE,
XSBench, RSBench, xz, DuckDB, OpenSSL, zstd, RocksDB, LevelDB, pigz, brotli, Rodinia others).

## Open cells and schedule

Every "—" and "⏳" above is planned. All open cells are timed with one compiler that carries all three PRs, so
the arms differ only by flags: stock, EA, DE, DE reads only, DE + EA.

| # | Cells | When |
|---|---|---|
| 1 | MySQL -O2: EA | ✅ done |
| 2 | memcached -O0: EA, four code layouts | ✅ done |
| 3 | Redis -O2: DE, DE reads only | ✅ done |
| 4 | Compiler with all three PRs; builds and relinks of every program; identical-code check | ✅ done (22 Oct) |
| 5 | Applications, -O0 and -O2: DE, DE reads only, DE + EA (Redis -O2 needs no leg) | running, until 23 Oct ~21:00 |
| 6 | Real applications from the screen: RocksDB, LevelDB, DuckDB, zstd, xz, pigz, brotli, OpenSSL; all arms, -O0 and -O2 | 24 Oct, until ~04:00 |
| 7 | Benchmark kernels: EA and DE + EA for NPB LU and BT (the other kernels have identical code) | 24 Oct, until ~06:00 |

Everything done: **24 Oct 2026, ~06:00**, provided each code-layout step runs without waiting for manual approval.

## Takeaways

- 🟢 **DE gives 1.8-3× on OpenMP kernels with repeated reads** (hotspot, LU, BT) and +6 % on ocean_ncp; no change
  elsewhere.
- **EA is measurable only at -O0** (SQLite on some tests, FFmpeg +0.4-0.9 %); at -O2 the code is identical or the
  speed unchanged.
- With one thread the saving is 7-26 %; the larger 8-thread gains are consistent with less cross-core traffic on
  TSan shadow memory (not confirmed at cache-line level).
- **DE, reads only** costs speed only where it removes fewer checks than DE: BT (0.83× of DE) and srad at 1 thread
  (0.97×).
