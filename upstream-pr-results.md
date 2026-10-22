# Upstream PR measurements

State of 22 Oct 2026. Figures are speed over stock TSan (>1 = faster), 95 % bootstrap interval in
brackets. A/A is a second stock run set and shows the noise. "No change" = the interval includes 1.

PRs: [#169896](https://github.com/llvm/llvm-project/pull/169896) (escape through local slots),
[#192945](https://github.com/llvm/llvm-project/pull/192945) (capture-tracking limit),
[#169897](https://github.com/llvm/llvm-project/pull/169897) (dominance-based elimination, DE).

## #192945 and #169896: the five applications

Wall clock, 8 runs per arm over 2-4 code layouts.

| App | Opt | A/A | #192945 alone | both PRs | #169896 over #192945 | Verdict |
|---|---|---|---|---|---|---|
| SQLite | -O0 | 0.967 [0.902, 1.053] | 1.056 [0.989, 1.131] | 1.079 [0.993, 1.184] | 1.021 [0.957, 1.085] | no change overall; checkpoint_starvation_1 1.77x, walthread1 1.07x |
| SQLite | -O2 | - | identical code | identical code | - | - |
| FFmpeg | -O0 | 1.000 [0.995, 1.006] | 1.009 [1.005, 1.015] | 1.004 [1.001, 1.009] | 0.995 [0.990, 1.000] | +0.9 % from the limit |
| FFmpeg | -O2 | 0.997 [0.993, 1.005] | - | 0.993 [0.976, 1.003] | - | no change |
| memcached | -O0 | 0.988 [0.958, 1.024] | identical code | 0.953 [0.937, 1.007]; repeat 0.983 [0.961, 1.008] | (= both) | unresolved |
| memcached | -O2 | - | identical code | identical code | - | - |
| Redis | -O0 | 0.998 [0.956, 1.005] | 0.999 [0.990, 1.005] | 0.996 [0.961, 1.004] | 0.998 [0.962, 1.007] | no change |
| Redis | -O2 | - | identical code | identical code | - | - |
| MySQL | -O0 | 1.000 [0.994, 1.007] | 1.001 [0.996, 1.007] | 0.999 [0.996, 1.006] | 0.999 [0.995, 1.003] | no change |
| MySQL | -O2 | | | running | | |

The slot rule (#169896) removes no access at -O1 or -O2.

## #169897 (DE): timed programs

Arms: de = DE on reads and writes; de-f1 = DE on reads only. Escape analysis off in both.

| Program | Removed checks de / de-f1 | Threads | A/A | de | de-f1 | Verdict |
|---|---|---|---|---|---|---|
| Rodinia hotspot | 25.5 / 25.5 % | 8 | 0.977 [0.823, 1.186] | 3.05 [2.61, 3.50] | 3.21 [2.79, 3.84] | faster |
| | | 1 | 0.996 [0.989, 1.004] | 1.10 [1.04, 1.16] | 1.10 [1.04, 1.16] | faster |
| NPB LU, class A | 28.9 / 28.8 % | 8 | 1.039 [0.937, 1.155] | 2.27 [2.03, 2.41] | 2.24 [2.00, 2.53] | faster |
| NPB LU, class W | 28.3 / 28.4 % | 8 | 0.960 [0.908, 1.049] | 2.02 [1.75, 2.20] | 2.04 [1.83, 2.23] | faster |
| NPB BT, class A | 50.7 / 37.1 % | 8 | 1.000 [0.994, 1.010] | 1.76 [1.75, 1.80] | 1.47 [1.46, 1.49] | faster; de-f1 0.83x of de |
| SPLASH-2x ocean_ncp | 9.9 / 9.9 % | 8 | 1.008 [0.991, 1.022] | 1.058 [1.046, 1.069] | 1.073 [1.062, 1.082] | faster (two legs) |
| SPLASH-2x water_nsquared | 25.2 / 25.2 % | 8 | 0.977 [0.931, 1.028] | 1.085 [1.009, 1.156] | 1.072 [1.007, 1.165] | unresolved (noise) |
| | | 1 | 1.000 [0.998, 1.002] | 1.26 [1.258, 1.263] | 1.26 [1.257, 1.262] | faster |
| SPLASH-2x water_spatial | 25.3 / 25.3 % | 8 | 0.959 [0.891, 1.021] | 1.104 [0.978, 1.194] | 1.080 [1.024, 1.153] | unresolved (noise) |
| | | 1 | 0.998 [0.996, 1.001] | 1.26 [1.258, 1.265] | 1.26 [1.259, 1.266] | faster |
| Rodinia srad | 13.9 / 10.7 % | 8 | 0.993 [0.885, 1.120] | 1.016 [0.922, 1.120] | 0.996 [0.909, 1.121] | no change |
| | | 1 | 0.999 [0.997, 1.003] | 1.074 [1.066, 1.083] | 1.041 [1.033, 1.050] | faster; de-f1 0.97x of de |
| PARSEC streamcluster | 11.2 / 2.0 % | 8 | 0.997 [0.983, 1.018] | 0.984 [0.959, 1.002] | 0.983 [0.964, 1.002] | no change |
| PARSEC fluidanimate | 3.2 / 2.6 % | 8 | 0.994 [0.970, 1.036] | 0.991 [0.954, 1.038] | 1.004 [0.970, 1.049] | no change |
| Rodinia lavaMD | 19.9 / 19.9 % | 8 | ~±10 % | 1.06-1.19 | 1.08-1.14 | dropped: A/A too wide |
| Redis -O2 | | | | | | pending |

Screened by removed checks (254 programs), not timed because below ~3 % or unresolved: NPB IS, SP,
FT, CG, MG, EP; GAP; PARSEC freqmine, canneal (crashes); SPLASH-2x fmm, barnes, radiosity, lu_cb,
lu_ncb, raytrace; pbbs; LULESH, HPCCG, CoMD, miniFE, XSBench, RSBench; xz, DuckDB, OpenSSL, zstd,
RocksDB, LevelDB, pigz, brotli; Rodinia backprop, pathfinder (cfd, heartwall: no input data).

## Takeaways

- #192945 / #169896: measurable only at -O0 (SQLite on some tests, FFmpeg +0.9 %); at -O2 the code
  is identical or unchanged in speed.
- #169897: 1.8-3x on OpenMP kernels with repeated reads (hotspot, LU, BT), +6 % on ocean_ncp,
  no change elsewhere.
- The 1-thread saving is 7-26 %; the larger 8-thread gains are consistent with less cross-core
  traffic on TSan shadow memory (not confirmed at cache-line level).
- de-f1 costs speed only where it removes fewer checks than de (BT, srad at 1 thread).
