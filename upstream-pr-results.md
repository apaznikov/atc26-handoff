# Upstream PR measurements

**All numbers are percentages against stock TSan.** Run time: speed-up, (stock time / arm time − 1) × 100, so
**+15 = 15 % faster**, **−2 = 2 % slower**. Compile time: change in compiler CPU time, + = slower to compile.

State of 26 Oct 2026: every Intel timing of the PRs as they stand is done; Chromium is building; DE + runtime fix and
the AMD check are pending ([Open items](#open-items)).

## Takeaways

- **On the 13 applications the PRs change nothing measurable in the overall score,** with three exceptions: xz -O0 DE and
  DE + EA +0.6 %, brotli -O0 DE + EA +1.8 % (🟢), and one Redis workload that DE makes **2.2 % slower** (LRANGE_600 at
  -O2, footnote ⁶; to be confirmed). SQLite -O0 gains up to +78 % on single tests (footnote ¹), but its overall score
  does not separate from noise.
- 🟢 **Kernels: DE gives +76 % to +205 % on OpenMP kernels with repeated reads** (hotspot, LU, BT) and +6 % on
  ocean_ncp; EA gives BT +8.5 %, DE + EA +109 %. Most other kernels are unresolved at 8 threads (the A/A of those legs
  spans ±3-30 %); they are being re-timed. At 1 thread DE gives barnes +4.9 %, srad +7 %, hotspot +10 %, water +26 %.
- **7-Zip benchmark** (multi-threaded): DE +1.1 %, DE + EA +1.7 %.
- **DE + runtime fix keeps DE's machine code,** so its speed should equal DE's except where race reports are frequent
  (SQLite, memcached and MySQL report races, MySQL ~12,000 per run); [why it is needed](#why-de-needs-a-runtime-fix).
- **Compile time:** within stock's own build-to-build spread except memcached DE + EA (+2.3 %, 0.2 s; its two builds
  differ by 3 %) and FFmpeg DE + EA (+0.3 % against a 0.2 % spread).
- **Race preservation (N = 10, every arm): no race site lost** on SQLite, memcached and MySQL at -O0 and -O2; the DE gap
  [below](#why-de-needs-a-runtime-fix) did not show up there. Redis and FFmpeg cannot test it: no build reports a race
  under their workloads.

## Columns

| Column | What it is | Upstream PR |
|---|---|---|
| **EA** | Escape analysis: capture-tracking limit + escape through local slots (not the paper's broader escape analysis) | [llvm#192945](https://github.com/llvm/llvm-project/pull/192945) + [llvm#169896](https://github.com/llvm/llvm-project/pull/169896) |
| **DE** | Dominance-based elimination of reads and writes, as in the PR today | [llvm#169897](https://github.com/llvm/llvm-project/pull/169897) |
| **DE + runtime fix** | DE plus a runtime that keeps a racing write after reporting it ([why it is needed](#why-de-needs-a-runtime-fix)) | llvm#169897 + a compiler-rt PR in preparation |
| **DE + EA** | DE + EA | all three |

Each arm's 95 % interval is compared with that of A/A (stock timed against itself, same leg):
🟢 faster (arm's lower bound above A/A's upper bound) · 🔴 slower (arm's upper bound below A/A's lower bound) ·
**=** no change (point estimate inside the A/A interval) · 🟡 unresolved (neither; the number is the point estimate) ·
† A/A wider than ±3 % ·
🟰 same machine code (EA as stock, DE + EA as DE), not timed · — not measured · ⏳ pending · ⏸ stopped

## Results

| Program | Threads | EA, % | DE, % | DE + runtime fix, % | DE + EA, % |
|---|:-:|:-:|:-:|:-:|:-:|
| **Applications, -O0** | | | | | |
| SQLite¹ † | 8 | 🟡 +8 | = | ⏳ | 🟡 +13 |
| FFmpeg | 8 | = | = | ⏳ | = |
| memcached⁵ † | 8 | = | = | ⏳ | = |
| Redis | 8 | = | = | ⏳ | = |
| MySQL | 8 | = | = | ⏳ | = |
| **Applications, -O2** | | | | | |
| SQLite² † | 8 | 🟰 | = | ⏳ | 🟰 |
| FFmpeg | 8 | = | = | ⏳ | = |
| memcached † | 8 | 🟰 | = | ⏳ | 🟰 |
| Redis⁶ | 8 | 🟰 | = | ⏳ | 🟰 |
| MySQL | 8 | = | = | ⏳ | = |
| **Real applications, -O2** (fixed work: db_bench --num, compressing a fixed file, OpenSSL enc + dgst) | | | | | |
| RocksDB db_bench³ † | 8 | 🟰 | = | ⏳ | 🟰 |
| LevelDB db_bench | 8 | 🟰 | = | ⏳ | 🟰 |
| DuckDB TPC-H | 8 | = | = | ⏳ | = |
| zstd | 8 | = | = | ⏳ | = |
| xz | 8 | 🟰 | = | ⏳ | 🟰 |
| pigz | 8 | 🟰 | = | ⏳ | 🟰 |
| brotli | 8 | 🟰 | = | ⏳ | 🟰 |
| OpenSSL | 8 | 🟡 +0.8 | = | ⏳ | = |
| **Real applications, -O0** | | | | | |
| RocksDB db_bench³ † | 8 | = | = | ⏳ | = |
| LevelDB db_bench | 8 | = | = | ⏳ | = |
| DuckDB TPC-H | 8 | = | = | ⏳ | = |
| zstd | 8 | = | = | ⏳ | = |
| xz | 8 | 🟡 +0.5 | 🟢 **+0.6** | ⏳ | 🟢 **+0.6** |
| pigz⁴ | 8 | 🟡 −2 | 🟡 −2 | ⏳ | 🟡 −2 |
| brotli | 8 | 🟡 +1.8 | = | ⏳ | 🟢 **+1.8** |
| OpenSSL | 8 | = | = | ⏳ | = |
| **Benchmark kernels** (NPB, Rodinia, SPLASH-2x, PARSEC: small programs that model one algorithm) | | | | | |
| Rodinia hotspot † | 8 | 🟰 | 🟢 **+205** | ⏳ | 🟰 |
| | 1 | 🟰 | 🟢 **+10** | ⏳ | 🟰 |
| NPB LU, class A † | 8 | — | 🟢 **+127** | ⏳ | — |
| NPB LU, class W † | 8 | = | 🟢 **+102** | ⏳ | 🟢 **+91** |
| NPB BT, class A † | 8 | 🟢 **+8.5** | 🟢 **+76** | ⏳ | 🟢 **+109** |
| SPLASH-2x ocean_ncp | 8 | 🟰 | 🟢 **+6** | ⏳ | 🟰 |
| SPLASH-2x water_nsquared † | 8 | 🟰 | 🟡 +9 | ⏳ | 🟰 |
| | 1 | 🟰 | 🟢 **+26** | ⏳ | 🟰 |
| SPLASH-2x water_spatial † | 8 | 🟰 | 🟡 +10 | ⏳ | 🟰 |
| | 1 | 🟰 | 🟢 **+26** | ⏳ | 🟰 |
| Rodinia srad † | 8 | 🟰 | = | ⏳ | 🟰 |
| | 1 | 🟰 | 🟢 **+7** | ⏳ | 🟰 |
| Rodinia lavaMD⁷ † | 8 | 🟰 | 🟡 +15 | ⏳ | 🟰 |
| SPLASH-2x barnes † | 8 | 🟰 | = | ⏳ | 🟰 |
| | 1 | 🟰 | 🟢 **+4.9** | ⏳ | 🟰 |
| PARSEC freqmine⁸ † | 8 | 🟰 | — | ⏳ | 🟰 |
| Rodinia particlefilter † | 8 | 🟰 | 🟡 +3.9 | ⏳ | 🟡 +4.9 |
| | 1 | 🟰 | = | ⏳ | = |
| SPLASH-2x radiosity | 8 | 🟰 | = | ⏳ | 🟰 |
| SPLASH-2x raytrace † | 8 | = | 🟡 −9 | ⏳ | = |
| | 1 | = | = | ⏳ | = |
| PARSEC streamcluster | 8 | 🟰 | = | ⏳ | 🟰 |
| PARSEC fluidanimate † | 8 | 🟰 | = | ⏳ | 🟰 |
| 7-Zip benchmark (llvm-test-suite; multi-threaded) | auto | 🟡 +0.4 | 🟢 **+1.1** | ⏳ | 🟢 **+1.7** |

¹ SQLite -O0 as a whole does not separate from noise (stress and trigger tests vary by up to 90 % between runs), but
single tests do: checkpoint_starvation_1 EA 🟢 **+78**, DE **+67**, DE + EA **+78**; walthread1 EA and DE + EA
🟢 **+7**; walthread2 EA 🟢 **+2.4**, DE + EA 🟢 **+3**. The DE and DE + EA figures for checkpoint_starvation_1 come from a
leg whose A/A for that test is itself off zero (+0.9 [+0.2, +1.5]); by the registered rule such a test is reported but
not called, although the effect is 70 times that offset.

² SQLite -O2: no single test separates from noise for DE (walthread1 +4 lies inside A/A); checkpoint_starvation_1
gains nothing at -O2.

³ RocksDB varies by ±3-5 % between runs at 8 threads. A re-time on 25 Oct with twice the runs (N = 8 per layout) did not
narrow it: -O2 A/A +2.5 [−2.2, +6.5], DE +5.5 [+1.0, +10.1]; -O0 every arm within A/A −2.3 [−4.5, +0.6]. The table
keeps the first leg.

⁴ pigz -O0: all four non-stock builds are 2 % slower in both code layouts, by the same amount whatever they remove, which
points to a shared build or layout effect rather than to the checks. The interval reaches the A/A interval, so unresolved.

⁵ memcached -O0 EA: the leg with four code layouts gives =; two earlier legs gave 🟡 −1.7 and 🟡 −4.7.

⁶ Redis -O2 DE: the composite is unchanged, but one workload is **🔴 slower**: LRANGE_600 −2.2 [−3.4, −1.0] against A/A
+0.2 [−0.9, +0.9]. LRANGE_300, LRANGE_500 and SADD point the same way (−2.0, −2.1, −5.3), within noise. At -O0, EA makes
LRANGE_300/500/600 🟢 faster by 1.9-2.1 %. To be confirmed by a re-run.

⁷ lavaMD DE: three legs disagree, +19 (unresolved), +6 (=) and +15 (unresolved, shown).

⁸ freqmine: the A/A of this leg is itself +13 [−2, +30], so the leg cannot be read; to be re-timed.

† The A/A interval of this row is wider than ±3 %: changes smaller than that cannot be resolved here.

**Selection.** 254 programs were screened by the share of checks DE removes, and the candidates then by the share of
executed accesses it removes. The real applications were timed regardless; kernels were timed above ~3 % of checks or
~4-5 % of accesses. Single-threaded programs (most of llvm-test-suite, the Abseil benchmarks) were timed too but are not
reported: a race detector is not run on single-threaded programs in practice. Not timed: NPB IS/SP/FT/CG/MG/EP, GAP, the other PARSEC, SPLASH-2x
and Rodinia kernels, pbbs, LULESH, HPCCG, miniFE, XSBench. In threaded programs, spin loops make the access count vary
by 13-129 % between runs.

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
(NPB BT +47 instead of +76).

## Compile time

CPU time of the compiler (user + sys summed over every source compile, configure probes excluded), -O2, 40 jobs,
change against stock in %. "Source files compiled" counts the compiler invocations that compile a source file (SQLite is
one amalgamated file, sqlite3.c). Two builds per arm (MySQL one), mean shown; the spread column is stock's difference between its two builds.
DE + runtime fix compiles as DE.

| Program | Source files compiled | Stock: compiler CPU time | Stock spread between builds, % | EA, % | DE, % | DE + EA, % |
|---|--:|--:|--:|:-:|:-:|:-:|
| SQLite | 1 | 16.4 s | 0.4 | +0.4 | +0.3 | +0.5 |
| memcached | 61 | 7.8 s | 0.9 | +0.1 | +0.8 | +2.3 |
| Redis | 140 | 28.5 s | 0.5 | −0.3 | +0.6 | −0.3 |
| FFmpeg | 1,936 | 350 s | 0.2 | −0.1 | +0.3 | +0.3 |
| MySQL | 3,662 | 4,270 s | (one build) | 0.0 | +0.2 | +0.1 |

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
| Resolve 🟡 and † cells, fill gaps (—): kernels at 8 threads after an A/A pilot per kernel, NPB LU A EA and DE + EA, SQLite -O0, RocksDB, the small 🟡 application cells, confirm the Redis -O2 LRANGE_600 slowdown | ⏳ queued after DE + runtime fix: ~50 h of machine time, Redis first, kernels after an A/A pilot each; expected to end ~30 Oct |
| Chromium (Intel): stock against DE + EA, Speedometer 3, blink_perf.dom and .paint | ⏳ current tree Chromium 156.0.8078.47 (Stable; pins clang 24, so no overrides), lld from the same LLVM commit; sync from 26 Oct 13:32, results expected 27 Oct ~01:00-03:00; the Aug-2024 tree (127) does not build with clang 24 without source edits. In that tree's libbase.so, DE + EA removes 1.74 % of read/write call sites |

Done 20-26 Oct: the applications at -O0 and -O2, the eight real applications, the kernels, the corpus
programs, the RocksDB re-time, compile time and race preservation.

## Method

- All timings: **focs**, Intel Xeon w9-3495X (56 cores, 112 threads, max 4.8 GHz), 250 GB RAM, Linux 6.8.
- Each run is pinned to one half of the machine (CPUs 4-27, 60-83; 8-thread runs use 4-11, 1-thread runs CPU 4); a run
  is discarded if the other half is busy. Every arm is timed in 2-4 code layouts and the results pooled.
- From 22 Oct on, one compiler carries all three PRs, so the arms differ only by flags. On the kernels, DE + EA and DE
  come from different legs: compare each with stock, not with each other.
- The corpus programs timed from 25 Oct (barnes, freqmine, particlefilter, radiosity, raytrace, 7-Zip) are built at -O2.
- DE and DE + EA are the PRs as they stand, without the runtime fix. DE + runtime fix compiles to DE's code; only the
  runtime differs, and only after a race report.
- Removed-check counts (screening) were taken on apollo (AMD EPYC 9115) and are not timings.
