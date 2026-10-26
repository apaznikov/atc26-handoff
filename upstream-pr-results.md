# Upstream PR measurements

**All numbers are percentages against stock TSan.** Run time: speed-up, (stock time / arm time − 1) × 100, so
**+15 = 15 % faster**, **−2 = 2 % slower**. Compile time: change in compiler CPU time, + = slower to compile.

State of 26 Oct 2026: every Intel timing of the PRs as they stand is done; Chromium is building; DE + runtime fix and
the AMD check are pending ([Open items](#open-items)).

## Takeaways

- **On the 13 applications the PRs change nothing measurable:** everything is within ±2 % of stock, except SQLite -O0
  on single tests (checkpoint_starvation_1 DE +67 %, DE + EA +78 %), xz -O0 +0.6 %, brotli -O0 DE + EA +1.8 % and
  FFmpeg -O0 EA +0.4 %.
- 🟢 **Kernels: DE gives +76 % to +205 % on OpenMP kernels with repeated reads** (hotspot, LU, BT) and +6 % on
  ocean_ncp; EA gives BT +8.5 %, DE + EA +109 %. At 1 thread DE gives barnes +4.9 %, srad +7 %, hotspot +10 %,
  water_nsquared and water_spatial +26 % (barnes, srad and water are unresolved or unchanged at 8 threads); raytrace and particlefilter
  stay unchanged.
- 🟢 **llvm-test-suite: DE is faster on 18 of 21 programs**, +1.1 % to +15.5 % (consumer-typeset +15.5 %, SIBsim4 +15.1 %,
  LoopRestructuring-flt +11.2 %, uuencode +9.1 %), median +3.8 %; enc-pc1 is unresolved, fourinarow and pcompress2
  unchanged. EA alone changes the code of 7zip and oggenc only, without a measurable effect.
- **DE + runtime fix keeps DE's machine code,** so its speed should equal DE's except where race reports are frequent
  (SQLite, memcached and MySQL report races, MySQL ~12,000 per run); [why it is needed](#why-de-needs-a-runtime-fix).
- **No arm adds measurable compile time:** at most +0.4 % on FFmpeg and MySQL; memcached's +2.6 % is 0.2 s.
- **Race preservation (N = 10, every arm): no race site lost** on SQLite, memcached and MySQL at -O0 and -O2; the DE gap
  [below](#why-de-needs-a-runtime-fix) did not show up there. Redis and
  FFmpeg cannot test it: no build reports a race under their workloads.

## Columns

| Column | What it is | Upstream PR |
|---|---|---|
| **EA** | Escape analysis: capture-tracking limit + escape through local slots (not the paper's broader escape analysis) | [llvm#192945](https://github.com/llvm/llvm-project/pull/192945) + [llvm#169896](https://github.com/llvm/llvm-project/pull/169896) |
| **DE** | Dominance-based elimination of reads and writes, as in the PR today | [llvm#169897](https://github.com/llvm/llvm-project/pull/169897) |
| **DE + runtime fix** | DE plus a runtime that keeps a racing write after reporting it ([why it is needed](#why-de-needs-a-runtime-fix)) | llvm#169897 + a compiler-rt PR in preparation |
| **DE + EA** | DE + EA | all three |

🟢 faster than A/A (stock timed against itself) · ⚪ no change · 🟡 unresolved (noise) · 🔴 slower ·
🟰 same machine code (EA as stock, DE + EA as DE), not timed · — not measured · ⏳ pending · ⏸ stopped

## Results

| Program | Threads | EA, % | DE, % | DE + runtime fix, % | DE + EA, % |
|---|:-:|:-:|:-:|:-:|:-:|
| **Applications, -O0** | | | | | |
| SQLite¹ | 8 | 🟡 +8 | 🟡 +8 | ⏳ | 🟡 +13 |
| FFmpeg | 8 | 🟢 **+0.4** | ⚪ +0.2 | ⏳ | ⚪ +0.2 |
| memcached | 8 | ⚪ −1 | ⚪ +0.4 | ⏳ | ⚪ +0.7 |
| Redis | 8 | ⚪ −0.4 | ⚪ +0.7 | ⏳ | ⚪ +0.9 |
| MySQL | 8 | ⚪ −0.1 | ⚪ −0.2 | ⏳ | ⚪ −0.3 |
| **Applications, -O2** | | | | | |
| SQLite² | 8 | 🟰 | ⚪ −2 | ⏳ | 🟰 |
| FFmpeg | 8 | ⚪ −0.7 | ⚪ −0.6 | ⏳ | ⚪ 0.0 |
| memcached | 8 | 🟰 | ⚪ −2 | ⏳ | 🟰 |
| Redis | 8 | 🟰 | ⚪ −0.3 | ⏳ | 🟰 |
| MySQL | 8 | ⚪ −0.3 | ⚪ −0.2 | ⏳ | ⚪ +0.3 |
| **Real applications, -O2** (fixed work: db_bench --num, compressing a fixed file, OpenSSL enc + dgst) | | | | | |
| RocksDB db_bench³ | 8 | 🟰 | 🟡 +1 | ⏳ | 🟰 |
| LevelDB db_bench | 8 | 🟰 | ⚪ −0.2 | ⏳ | 🟰 |
| DuckDB TPC-H | 8 | ⚪ +0.1 | ⚪ +0.2 | ⏳ | ⚪ −0.1 |
| zstd | 8 | ⚪ 0.0 | ⚪ +0.3 | ⏳ | ⚪ 0.0 |
| xz | 8 | 🟰 | ⚪ +0.3 | ⏳ | 🟰 |
| pigz | 8 | 🟰 | ⚪ +0.1 | ⏳ | 🟰 |
| brotli | 8 | 🟰 | ⚪ +0.5 | ⏳ | 🟰 |
| OpenSSL | 8 | ⚪ +0.8 | ⚪ +0.5 | ⏳ | ⚪ −0.2 |
| **Real applications, -O0** | | | | | |
| RocksDB db_bench³ | 8 | 🟡 +2 | 🟡 +1 | ⏳ | 🟡 +1 |
| LevelDB db_bench | 8 | ⚪ 0.0 | ⚪ +0.8 | ⏳ | ⚪ +0.6 |
| DuckDB TPC-H | 8 | ⚪ −0.3 | ⚪ +0.1 | ⏳ | ⚪ +0.3 |
| zstd | 8 | ⚪ −1.3 | ⚪ +0.4 | ⏳ | ⚪ +0.6 |
| xz | 8 | ⚪ +0.5 | 🟢 **+0.6** | ⏳ | 🟢 **+0.6** |
| pigz⁴ | 8 | ⚪ −2 | ⚪ −2 | ⏳ | ⚪ −2 |
| brotli | 8 | 🟡 +1.8 | ⚪ −0.1 | ⏳ | 🟢 **+1.8** |
| OpenSSL | 8 | ⚪ −0.4 | ⚪ −0.2 | ⏳ | ⚪ 0.0 |
| **Benchmark kernels** (NPB, Rodinia, SPLASH-2x, PARSEC: small programs that model one algorithm) | | | | | |
| Rodinia hotspot | 8 | 🟰 | 🟢 **+205** | ⏳ | 🟰 |
| | 1 | 🟰 | 🟢 **+10** | ⏳ | 🟰 |
| NPB LU, class A | 8 | — | 🟢 **+127** | ⏳ | — |
| NPB LU, class W | 8 | 🟡 −4 | 🟢 **+102** | ⏳ | 🟢 **+91** |
| NPB BT, class A | 8 | 🟢 **+8.5** | 🟢 **+76** | ⏳ | 🟢 **+109** |
| SPLASH-2x ocean_ncp | 8 | 🟰 | 🟢 **+6** | ⏳ | 🟰 |
| SPLASH-2x water_nsquared | 8 | 🟰 | 🟡 +9 | ⏳ | 🟰 |
| | 1 | 🟰 | 🟢 **+26** | ⏳ | 🟰 |
| SPLASH-2x water_spatial | 8 | 🟰 | 🟡 +10 | ⏳ | 🟰 |
| | 1 | 🟰 | 🟢 **+26** | ⏳ | 🟰 |
| Rodinia srad | 8 | 🟰 | ⚪ +2 | ⏳ | 🟰 |
| | 1 | 🟰 | 🟢 **+7** | ⏳ | 🟰 |
| Rodinia lavaMD | 8 | 🟰 | 🟡 ~+10 | ⏳ | 🟰 |
| SPLASH-2x barnes | 8 | 🟰 | 🟡 +8 | ⏳ | 🟰 |
| | 1 | 🟰 | 🟢 **+4.9** | ⏳ | 🟰 |
| PARSEC freqmine | 8 | 🟰 | 🟡 +5 | ⏳ | 🟰 |
| Rodinia particlefilter | 8 | 🟰 | 🟡 +3.9 | ⏳ | 🟡 +4.9 |
| | 1 | 🟰 | ⚪ 0.0 | ⏳ | ⚪ −0.4 |
| SPLASH-2x radiosity | 8 | 🟰 | ⚪ +0.2 | ⏳ | 🟰 |
| SPLASH-2x raytrace | 8 | 🟡 +2 | 🟡 −9 | ⏳ | 🟡 −7 |
| | 1 | ⚪ −0.1 | ⚪ −0.1 | ⏳ | ⚪ −0.2 |
| PARSEC streamcluster | 8 | 🟰 | ⚪ −2 | ⏳ | 🟰 |
| PARSEC fluidanimate | 8 | 🟰 | ⚪ −1 | ⏳ | 🟰 |
| **llvm-test-suite** (single-threaded except 7zip) | | | | | |
| consumer-typeset | 1 | 🟰 | 🟢 **+15.5** | ⏳ | 🟰 |
| SIBsim4 | 1 | 🟰 | 🟢 **+15.1** | ⏳ | 🟰 |
| LoopRestructuring-flt | 1 | 🟰 | 🟢 **+11.2** | ⏳ | 🟰 |
| uuencode | 1 | 🟰 | 🟢 **+9.1** | ⏳ | 🟰 |
| uudecode | 1 | 🟰 | 🟢 **+6.0** | ⏳ | 🟰 |
| Olden voronoi | 1 | 🟰 | 🟢 **+5.6** | ⏳ | 🟰 |
| CLAMR | 1 | 🟰 | 🟢 **+5.5** | ⏳ | 🟰 |
| espresso | 1 | 🟰 | 🟢 **+5.4** | ⏳ | 🟰 |
| Olden bisort | 1 | 🟰 | 🟢 **+4.4** | ⏳ | 🟰 |
| oggenc | 1 | ⚪ 0.0 | 🟢 **+4.1** | ⏳ | 🟢 **+4.3** |
| CoMD | 1 | 🟰 | 🟢 **+3.8** | ⏳ | 🟰 |
| LoopRestructuring-dbl | 1 | 🟰 | 🟢 **+3.7** | ⏳ | 🟰 |
| Obsequi | 1 | 🟰 | 🟢 **+3.4** | ⏳ | 🟰 |
| enc-pc1 | 1 | 🟰 | 🟡 +2.5 | ⏳ | 🟰 |
| g721encode | 1 | 🟰 | 🟢 **+2.4** | ⏳ | 🟰 |
| Olden power | 1 | 🟰 | 🟢 **+2.4** | ⏳ | 🟰 |
| RSBench | 1 | 🟰 | 🟢 **+2.0** | ⏳ | 🟰 |
| mpeg2decode | 1 | 🟰 | 🟢 **+1.3** | ⏳ | 🟰 |
| 7zip-benchmark | auto | 🟡 +0.4 | 🟢 **+1.1** | ⏳ | 🟢 **+1.7** |
| fourinarow | 1 | 🟰 | ⚪ +0.1 | ⏳ | 🟰 |
| pcompress2 | 1 | 🟰 | ⚪ 0.0 | ⏳ | 🟰 |
| **Abseil benchmarks** | | | | | |
| graphcycles_benchmark | 1 | 🟰 | 🟡 +0.9 | ⏳ | 🟰 |
| str_join_benchmark | 1 | 🟰 | 🟡 −1.6 | ⏳ | 🟰 |

¹ SQLite -O0 as a whole does not separate from noise (stress and trigger tests vary by up to 90 % between runs), but
single tests do: checkpoint_starvation_1 EA 🟢 **+77**, DE 🟢 **+67**, DE + EA 🟢 **+78**; walthread1 EA and DE + EA
🟢 **+7**; walthread2 DE + EA 🟢 **+3**.

² SQLite -O2: no single test separates from noise for DE (walthread1 +4, unresolved); checkpoint_starvation_1 gains
nothing at -O2.

³ RocksDB varies by ±3-5 % between runs at 8 threads. A re-time on 25 Oct with twice the runs (N = 8 per layout) did not
narrow it: -O2 A/A +2.5 [−2.2, +6.5], DE +5.5 [+1.0, +10.1]; -O0 every arm within A/A −2.3 [−4.5, +0.6]. The table
keeps the first leg.

⁴ pigz -O0: all four non-stock builds are 2 % slower in both code layouts, by the same amount whatever they remove, which
points to a shared build or layout effect rather than to the checks; not called slower.

**Selection.** 254 programs were screened by the share of checks DE removes, and the candidates then by the share of
executed accesses it removes. The real applications were timed regardless; kernels and llvm-test-suite programs were
timed above ~3 % of checks or ~4-5 % of accesses. Not timed: NPB IS/SP/FT/CG/MG/EP, GAP, the other PARSEC, SPLASH-2x
and Rodinia kernels, pbbs, LULESH, HPCCG, miniFE, XSBench. In threaded programs, spin loops make the access count vary
by 13-129 % between runs. The share of accesses predicts the direction but not the size: consumer-typeset 19.7 % gives
+15.5 %, uuencode 29.4 % gives +9.1 %, fourinarow 30.7 % gives nothing.

## Why DE needs a runtime fix

Stock TSan forgets the write that raced: after a report it clears the location's history and does not record the
access ([`DoReportRace`](https://github.com/llvm/llvm-project/blob/main/compiler-rt/lib/tsan/rtl/tsan_rtl_access.cpp)).
Later reads of the location are then skipped altogether; the next write is recorded again. DE removes that next write
when an earlier write covers it, so nothing is recorded, and a later race on the location disappears.

```c
int x;
// Thread 2:            // Thread 1:
x = 7;    // C1
                        x = 1;   // A: races with C1 → report; history of x cleared, A not recorded
                        x = 2;   // B: covered by A
x = 9;    // C2
```

| Detector | Reports | C2 visible? |
|---|---|---|
| Stock TSan | A–C1, B–C2 (B is recorded, C2 races with it) | yes |
| TSan + DE (llvm#169897 today) | A–C1 only (B's check removed, the history stays empty) | **no** |
| TSan + DE + runtime fix | A–C1, A–C2 (A is recorded after its report) | yes |

The paper's argument allows reporting A–C2 instead of B–C2 (B's race is dominated by A's); it does not allow losing C2.
The paper's runtime has this fix (`__tsan_write*_keep` for covering writes). Checked on 26 Oct with the paper's
regression test for this case: the paper compiler with DE reports 2 races, the PR compiler with DE reports 1.
Removing only reads would also close the gap, but it costs much of DE's gain where the removed writes matter
(BT +47 instead of +76, consumer-typeset +8.9 instead of +15.5).

## Compile time

CPU time of the compiler (user + sys summed over every source compile, configure probes excluded), -O2, 40 jobs,
change against stock in %. Stock varies by 0.2-0.6 % between builds. DE + runtime fix compiles as DE.

| Program | Compiles | Stock, CPU time | EA, % | DE, % | DE + EA, % |
|---|--:|--:|:-:|:-:|:-:|
| SQLite | 1 | 16.4 s | +0.6 | +0.3 | +0.6 |
| memcached | 61 | 7.8 s | +0.7 | +1.3 | +2.6 |
| Redis | — | 28.5 s | −0.3 | +0.5 | −0.2 |
| FFmpeg | 1,936 | 350 s | −0.1 | +0.3 | +0.3 |
| MySQL | 3,662 | 4,270 s | 0.0 | +0.2 | +0.1 |

## Race preservation

Each arm is run N = 10 times beside its own stock arm (the five arms in parallel; CPU sets rotated across runs from memcached -O2 on, and in a re-run of memcached -O0) and
compared race site by race site (location and writer site). Kept = the arm reports a site that stock reports;
undetermined = stock reports the site too rarely (≤ 2 of 10 runs) for an absence in the arm to mean anything.

| Program | Level | EA | DE | DE + runtime fix | DE + EA | Notes |
|---|:-:|:-:|:-:|:-:|:-:|---|
| SQLite | -O0 | 0 lost | 0 lost | ⏳ | 0 lost | 4 sites kept; walIndexRecover undetermined (stock 1/10) |
| SQLite | -O2 | 0 lost | 0 lost | ⏳ | 0 lost | 2 sites kept; walIndexRecover undetermined (stock 1/10); the two walIndexWriteHdr sites of -O0 are absent at -O2 in every build, stock included |
| memcached | -O0 | 0 lost | 0 lost | ⏳ | 0 lost | 4 sites kept at 10/10; the scheduling-dependent conn_new/conn_close family (7 sites) is undetermined (stock 2/10) |
| memcached | -O2 | 0 lost | 0 lost | ⏳ | 0 lost | all 11 sites kept: 4 at 10/10, the conn_new/conn_close family 2-3/10 in every arm, stock included |
| MySQL | -O0 | 0 lost | 0 lost | ⏳ | 0 lost | ~12,100 reports per run in every arm; 251 sites kept; 13 undetermined |
| MySQL | -O2 | 0 lost | 0 lost | ⏳ | 0 lost | ~12,000 reports per run; 226 sites kept; 26 undetermined. Closest case: dict_stats_set_persistent, stock 4/10, DE 0/10, but DE + EA, which removes every check DE removes, sees it (1/10), so DE's 0/10 is detection noise |
| Redis, FFmpeg | -O0 | — | — | — | — | no test: no build, stock included, reports any race under these workloads (0 reports in 50 runs each); -O2 skipped |

## Open items

| Item | State |
|---|---|
| AMD (apollo): NPB BT A, LU W, hotspot, SQLite -O0 | ⏳ waits for a quiet window on apollo |
| DE + runtime fix: compiler-rt PR (port of the paper runtime's keep-the-racing-write step), then timing and race preservation | ⏳ in preparation |
| Chromium (Intel): stock against DE + EA, Speedometer 3, blink_perf.dom and .paint | ⏳ current tree Chromium 156.0.8078.47 (Stable; pins clang 24, so no overrides), lld from the same LLVM commit; sync from 26 Oct 13:32, results expected 27 Oct ~01:00-03:00; the Aug-2024 tree (127) does not build with clang 24 without source edits. In that tree's libbase.so, DE + EA removes 1.74 % of read/write call sites |

Done 20-26 Oct: the applications at -O0 and -O2, the eight real applications, the kernels, the llvm-test-suite and
Abseil programs, the RocksDB re-time, compile time and race preservation.

## Method

- All timings: **focs**, Intel Xeon w9-3495X (56 cores, 112 threads, max 4.8 GHz), 250 GB RAM, Linux 6.8.
- Each run is pinned to one half of the machine (CPUs 4-27, 60-83; 8-thread runs use 4-11, 1-thread runs CPU 4); a run
  is discarded if the other half is busy. Every arm is timed in 2-4 code layouts and the results pooled.
- From 22 Oct on, one compiler carries all three PRs, so the arms differ only by flags. On the kernels, DE + EA and DE
  come from different legs: compare each with stock, not with each other.
- The corpus programs timed from 25 Oct (barnes, freqmine, particlefilter, radiosity, raytrace, llvm-test-suite, Abseil) are built at -O2.
- DE and DE + EA are the PRs as they stand, without the runtime fix. DE + runtime fix compiles to DE's code; only the
  runtime differs, and only after a race report.
- Removed-check counts (screening) were taken on apollo (AMD EPYC 9115) and are not timings.
