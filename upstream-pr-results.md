# Upstream PR measurements

> Speed-up over stock TSan, **>1 = faster**. State of 25 Oct 2026: complete; RocksDB re-timed (⁵).

| Column | What it is | Upstream PR |
|---|---|---|
| **EA** | Escape analysis: capture-tracking limit + escape through local slots | [llvm#192945](https://github.com/llvm/llvm-project/pull/192945) + [llvm#169896](https://github.com/llvm/llvm-project/pull/169896) |
| **DE** | Dominance-based elimination, reads and writes | [llvm#169897](https://github.com/llvm/llvm-project/pull/169897) |
| **DE, reads only** | DE that keeps every write checked (fix for the write-after-report gap) | [llvm#169897](https://github.com/llvm/llvm-project/pull/169897) with F1 |
| **DE + EA** | DE + capture limit + local slots | all three |

🟢 faster  ·  ⚪ no change  ·  🟡 unresolved (noise)  ·  🔴 slower  ·  🟰 identical code (EA: to stock; DE + EA: to DE), not timed  ·  — not measured  ·  ⏳ in progress

## Machine

All timings: **focs**, Intel Xeon w9-3495X (56 cores, 112 threads, max 4.8 GHz), 250 GB RAM, Linux 6.8.
Each run is pinned to one half of the machine (CPUs 4-27, 60-83; 8-thread runs use 4-11, 1-thread runs CPU 4); a
run is discarded if the other half is busy. Every arm is timed in 2-4 code layouts and the results pooled.
Removed-check counts (screening) were taken on apollo (AMD EPYC 9115) and are not timings.

## Results

| Program | Threads | EA | DE | DE, reads only | DE + EA |
|---|:-:|:-:|:-:|:-:|:-:|
| **Applications, -O0** | | | | | |
| SQLite | 8 | ⚪ 1.08¹ | 🟡 1.08² | 🟡 1.05² | 🟡 1.13² |
| FFmpeg | 8 | 🟢 **1.004** | ⚪ 1.002 | ⚪ 1.003 | ⚪ 1.002 |
| memcached | 8 | ⚪ 0.99 | ⚪ 1.004 | ⚪ 1.010 | ⚪ 1.007 |
| Redis | 8 | ⚪ 0.996 | ⚪ 1.007 | ⚪ 1.003 | ⚪ 1.009 |
| MySQL | 8 | ⚪ 0.999 | ⚪ 0.998 | ⚪ 0.998 | ⚪ 0.997 |
| **Applications, -O2** | | | | | |
| SQLite | 8 | 🟰 1.000 | ⚪ 0.98³ | ⚪ 0.98³ | 🟰 0.98 |
| FFmpeg | 8 | ⚪ 0.993 | ⚪ 0.994 | ⚪ 0.996 | ⚪ 1.000 |
| memcached | 8 | 🟰 1.000 | ⚪ 0.98 | ⚪ 0.99 | 🟰 0.98 |
| Redis | 8 | 🟰 1.000 | ⚪ 0.997 | ⚪ 0.997 | 🟰 0.997 |
| MySQL | 8 | ⚪ 0.997 | ⚪ 0.998 | ⚪ 1.002 | ⚪ 1.003 |
| **Real applications, -O2** (fixed work: db_bench --num, compressing a fixed file, OpenSSL enc + dgst) | | | | | |
| RocksDB db_bench⁵ | 8 | 🟰 1.000 | 🟡 1.01 | 🟡 1.02 | 🟰 1.01 |
| LevelDB db_bench | 8 | 🟰 1.000 | ⚪ 0.998 | ⚪ 0.995 | 🟰 0.998 |
| DuckDB TPC-H | 8 | ⚪ 1.001 | ⚪ 1.002 | ⚪ 0.995 | ⚪ 0.999 |
| zstd | 8 | ⚪ 1.000 | ⚪ 1.003 | ⚪ 0.998 | ⚪ 1.000 |
| xz | 8 | 🟰 1.000 | ⚪ 1.003 | ⚪ 0.998 | 🟰 1.003 |
| pigz | 8 | 🟰 1.000 | ⚪ 1.001 | ⚪ 0.994 | 🟰 1.001 |
| brotli | 8 | 🟰 1.000 | ⚪ 1.005 | ⚪ 0.993 | 🟰 1.005 |
| OpenSSL | 8 | ⚪ 1.008 | ⚪ 1.005 | ⚪ 1.001 | ⚪ 0.998 |
| **Real applications, -O0** | | | | | |
| RocksDB db_bench⁵ | 8 | 🟡 1.02 | 🟡 1.01 | 🟡 1.06 | 🟡 1.01 |
| LevelDB db_bench | 8 | ⚪ 1.000 | ⚪ 1.008 | ⚪ 1.010 | ⚪ 1.006 |
| DuckDB TPC-H | 8 | ⚪ 0.997 | ⚪ 1.001 | ⚪ 1.003 | ⚪ 1.003 |
| zstd | 8 | ⚪ 0.987 | ⚪ 1.004 | ⚪ 1.002 | ⚪ 1.006 |
| xz | 8 | ⚪ 1.005 | 🟢 **1.006** | ⚪ 1.003 | 🟢 **1.006** |
| pigz | 8 | ⚪ 0.98⁴ | ⚪ 0.98⁴ | ⚪ 0.98⁴ | ⚪ 0.98⁴ |
| brotli | 8 | ⚪ 1.018 | ⚪ 0.999 | ⚪ 0.997 | 🟢 **1.018** |
| OpenSSL | 8 | ⚪ 0.996 | ⚪ 0.998 | ⚪ 0.998 | ⚪ 1.000 |
| **Benchmark kernels** (NPB, Rodinia, SPLASH-2x, PARSEC: small programs that model one algorithm, not applications) | | | | | |
| Rodinia hotspot | 8 | 🟰 1.000 | 🟢 **3.05** | 🟢 **3.21** | 🟰 3.05 |
| | 1 | 🟰 1.000 | 🟢 **1.10** | 🟢 **1.10** | 🟰 1.10 |
| NPB LU, class A | 8 | — | 🟢 **2.27** | 🟢 **2.24** | — |
| NPB LU, class W | 8 | 🟡 0.96 | 🟢 **2.02** | 🟢 **2.04** | 🟢 **1.91** |
| NPB BT, class A | 8 | 🟢 **1.085** | 🟢 **1.76** | 🟢 **1.47** | 🟢 **2.09** |
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
| **Corpus programs timed from 26 Oct** (schedule row 11; -O2) | | | | | |
| SPLASH-2x barnes | 8 | 🟰 1.000 | 🟡 1.08 | 🟡 1.07 | 🟰 1.08 |
| SPLASH-2x radiosity | 8 | 🟰 1.000 | ⚪ 1.002 | ⚪ 1.002 | 🟰 1.002 |
| PARSEC freqmine | 8 | 🟰 1.000 | 🟡 1.05 | 🟡 1.08 | 🟰 1.05 |
| llvm-test-suite SIBsim4 (single-threaded) | 1 | 🟰 1.000 | 🟢 **1.151** | 🟢 **1.154** | 🟰 1.151 |
| Abseil graphcycles_benchmark | 1 | 🟰 1.000 | 🟡 1.009 | 🟡 1.009 | 🟰 1.009 |
| Abseil str_join_benchmark | 1 | 🟰 1.000 | 🟡 0.984 | 🟡 0.990 | 🟰 0.984 |
| llvm-test-suite CLAMR (single-threaded) | 1 | 🟰 1.000 | 🟢 **1.055** | 🟢 **1.057** | 🟰 1.055 |
| llvm-test-suite 7zip-benchmark | auto | 🟡 1.004 | 🟢 **1.011** | 🟢 **1.016** | 🟢 **1.017** |
| llvm-test-suite enc-pc1 (single-threaded) | 1 | 🟰 1.000 | 🟡 1.025 | ⚪ 0.999 | 🟰 1.025 |
| SPLASH-2x raytrace | 8 | 🟡 1.02 | 🟡 0.91 | 🟡 0.99 | 🟡 0.93 |
| Rodinia particlefilter | 8 | 🟰 1.000 | 🟡 1.039 | 🟡 1.045 | 🟡 1.049 |

¹ SQLite -O0 as a whole shows no change, but two tests do: checkpoint_starvation_1 🟢 **1.77×**, walthread1 🟢 **1.07×**.

² SQLite -O0 as a whole does not separate from noise (stress and trigger tests vary by up to 1.9× between runs), but
three tests do: checkpoint_starvation_1 DE 🟢 **1.67×**, reads only 🟢 **1.35×**, DE + EA 🟢 **1.78×**;
walthread1 DE + EA 🟢 **1.07×**; walthread2 DE + EA 🟢 **1.03×**.

³ SQLite -O2: only walthread1 separates from noise, DE reads only 🟢 **1.06×** (DE 1.04×, unresolved);
checkpoint_starvation_1 gains nothing at -O2.

⁴ pigz -O0: all four non-stock builds are 2 % slower in both code layouts, by the same amount whatever they
remove, which points to a shared build or layout effect rather than to the checks; not called slower.

⁵ RocksDB re-timed on 25 Oct with twice the runs (N = 8 per layout, none discarded); the spread did not narrow,
since RocksDB varies by ±3-5 % between runs at 8 threads. -O2: A/A 1.025 [0.978, 1.065], DE 1.055 [1.010, 1.101],
reads only 1.065 [1.010, 1.110], both still 🟡. -O0: A/A 0.977 [0.955, 1.006], EA 1.005, DE 0.979, reads only
1.007, DE + EA 0.995, all within A/A. The table keeps the first leg; neither leg separates any arm from noise.

In every column, EA means the two EA PRs (capture limit + local slots); the paper's broader escape analysis is not used.

Screening: 254 programs were ranked by the share of checks DE removes. Kernels below ~3 % were not timed (NPB
IS/SP/FT/CG/MG/EP, GAP, PARSEC/SPLASH-2x others, pbbs, LULESH, HPCCG, CoMD, miniFE, XSBench, RSBench, Rodinia
others); the eight real applications were timed regardless.

## Race preservation

Each arm is run N = 10 times beside its own stock arm (the five arms in parallel) and compared race site by race
site (location and writer site). Kept = the arm reports a site that stock reports; undetermined = stock reports the
site in too few runs (here 1 of 10) for an absence in the arm to mean anything.

| Program | Level | EA | DE | DE, reads only | DE + EA | Notes |
|---|:-:|:-:|:-:|:-:|:-:|---|
| SQLite | -O0 | 0 lost | 0 lost | 0 lost | 0 lost | stock reports races in 10/10 runs; 4 sites kept in every arm (6-10/10 runs, as stock); undetermined: walIndexRecover (stock 1/10, every arm 0/10) |
| SQLite | -O2 | 0 lost | 0 lost | 0 lost | 0 lost | stock reports races in 8/10 runs; 2 sites kept in every arm (stock 7-8/10, arms 5-10/10); undetermined: walIndexRecover (stock 1/10, arms 0-4/10); the two walIndexWriteHdr sites of -O0 are absent at -O2 in every build, stock included |
| memcached | -O0 | 0 lost | 0 lost | 0 lost | 0 lost | stock reports races in 10/10 runs; 4 sites kept at 10/10 in every arm (clock_handler, do_item_link, do_item_unlink ×2); the scheduling-dependent conn_new/conn_close family (7 sites) appears only in EA, DE and reads only (5-7/10), not in stock or DE + EA: an artefact of each arm keeping one CPU set. Re-run with rotated sets: 4 sites kept at 10/10; the conn family is rare in every arm (stock 2/10, arms 0-2/10), undetermined |
| memcached | -O2 | 0 lost | 0 lost | 0 lost | 0 lost | CPU sets rotated across runs from here on; stock reports races in 10/10 runs; all 11 sites kept: 4 at 10/10 in every arm, the conn_new/conn_close family 2-3/10 in every arm, stock included |
| Redis | -O0 | — | — | — | — | no test: no build reports any race under this workload (0 reports in 50 runs, stock included); -O2 skipped for the same reason |
| FFmpeg | -O0 | — | — | — | — | no test: 0 reports in 50 runs, stock included (as in 30 runs in September; the encoders are uninstrumented system libraries); -O2 skipped |
| MySQL | -O0 | 0 lost | 0 lost | 0 lost | 0 lost | stock reports races in 10/10 runs (~12,100 reports per run in every arm); 251 sites kept; 13 undetermined (stock ≤ 2/10 each) |
| MySQL | -O2 | 0 lost | 0 lost | 0 lost | 0 lost | stock reports races in 10/10 runs (~12,000 reports per run in every arm); 226 sites kept; 26 undetermined, 15 of them seen only in arms; the closest case, dict_stats_set_persistent (stock 4/10, DE 0/10), is seen by DE + EA (1/10), which removes every check DE removes, so DE's 0/10 is detection noise |

## Schedule

All cells from 22 Oct on were timed with one compiler that carries all three PRs, so the arms differ only by flags:
stock, EA, DE, DE reads only, DE + EA. DE + EA and DE come from different legs on the kernels; compare each with
stock, not with each other.

| # | Cells | When |
|---|---|---|
| 1 | MySQL -O2: EA | ✅ done |
| 2 | memcached -O0: EA, four code layouts | ✅ done |
| 3 | Redis -O2: DE, DE reads only | ✅ done |
| 4 | Compiler with all three PRs; builds and relinks of every program; identical-code check | ✅ done (22 Oct) |
| 5 | Applications, -O0 and -O2: DE, DE reads only, DE + EA (Redis -O2 needs no leg) | ✅ done (23 Oct) |
| 6 | Real applications from the screen: RocksDB, LevelDB, DuckDB, zstd, xz, pigz, brotli, OpenSSL; all arms, -O0 and -O2 | ✅ done (24 Oct) |
| 7 | Benchmark kernels: EA and DE + EA for NPB LU and BT (the other kernels have identical code) | ✅ done (24 Oct) |
| 8 | RocksDB -O0 and -O2, all arms, twice the runs | ✅ done (25 Oct, 14:39) |
| 9 | Race preservation: 5 applications, -O0 and -O2, all arms, N = 10 | ✅ done (26 Oct, 01:15): no site lost; Redis and FFmpeg untestable (no build reports a race) |
| 10 | AMD (apollo): NPB BT A, LU W, hotspot, SQLite -O0 | ⏳ waits for a quiet window on apollo |
| 11 | Corpus programs not yet timed with a DE share of accesses above ~4 % in at least one count run (SPLASH-2x barnes, radiosity, raytrace; PARSEC freqmine; Rodinia particlefilter; Abseil str_join, graphcycles; llvm-test-suite SIBsim4, CLAMR, 7zip, enc-pc1). Only raytrace resolves over 3 count runs (4.4 %); in the threaded others, spin loops make the count vary by 13-129 % | ✅ done (26 Oct) |
| 12 | Compile time: 5 applications at -O2, every arm, compile step only (CPU time summed over the compiler invocations) | ⏳ running on focs, ~6 h |
| 13 | 1-thread legs: SPLASH-2x raytrace, Rodinia particlefilter (barnes if cheap) | ⏳ after row 12 |

Rows 1-7 finished **24 Oct 2026, 04:55**; rows 8-10 were added on 25 Oct. NPB LU class A was not timed with EA and DE + EA (one LU input suffices).

## Takeaways

- 🟢 **DE gives 1.8-3× on OpenMP kernels with repeated reads** (hotspot, LU, BT) and +6 % on ocean_ncp; no change
  elsewhere.
- 🟢 **Among the corpus programs timed on 26 Oct, DE speeds up three:** SIBsim4 1.15×, CLAMR 1.055× (both
  single-threaded) and 7zip +1.1 % (+1.7 % with EA). The threaded SPLASH-2x/PARSEC/Rodinia programs vary too much
  at 8 threads to separate (barnes, freqmine, raytrace, particlefilter); radiosity is unchanged.
- **Race preservation (N = 10, every arm): no race site lost** on SQLite, memcached and MySQL at -O0 and -O2
  (MySQL: 251 and 226 sites kept). Redis and FFmpeg cannot test it: no build, stock included, reports a race
  under their workloads.
- **On the 13 applications DE helps only SQLite at -O0** (checkpoint_starvation_1 1.67×, with EA 1.78×) and xz at
  -O0 (+0.6 %); everything else is within ±2 % of stock.
- **EA is measurable on applications only at -O0** (SQLite on some tests, FFmpeg +0.4-0.9 %, brotli +1.8 % with
  DE); at -O2 the code is mostly identical or the speed unchanged. On kernels EA gives NPB BT +8.5 %, and DE + EA
  2.09× there.
- With one thread the saving is 7-26 %; the larger 8-thread gains are consistent with less cross-core traffic on
  TSan shadow memory (not confirmed at cache-line level).
- **DE, reads only** costs speed only where it removes fewer checks than DE: BT (0.83× of DE) and srad at 1 thread
  (0.97×).
