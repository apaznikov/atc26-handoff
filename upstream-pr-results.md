# Upstream PR measurements

**All numbers are percentages against stock TSan.** Run time: speed-up, (stock time / arm time − 1) × 100, so
**+15 = 15 % faster**, **−2 = 2 % slower**. Compile time: change in compiler CPU time, + = slower to compile.

State of 26 Oct 2026: every Intel timing of the PRs as they stand is done; Chromium is building; DE + runtime fix and
the AMD check are pending ([Open items](#open-items)).

## Takeaways

- **On the 13 applications the PRs change nothing measurable in the overall score,** with three exceptions: xz -O0 DE and
  DE + EA +0.6 %, brotli -O0 DE + EA +1.8 % (🟢), and two slowdowns: Redis LRANGE at -O2 2-3 % slower with DE (footnote ⁶), and pigz -O0
  2.2 % slower with every arm, the same whatever is removed (footnote ⁴). SQLite -O0 gains up to +78 % on single tests (footnote ¹), but its overall score
  does not separate from noise.
- 🟢 **Kernels: DE gives +76 % to +205 % on OpenMP kernels with repeated reads** (hotspot, LU, BT) and +6 % on
  ocean_ncp; EA gives BT +8.5 %, DE + EA +109 %. Most other kernels are unresolved at 8 threads (the A/A of those legs
  spans ±3-30 %); they are being re-timed.
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
| **DE + runtime fix** | DE plus a runtime that keeps a racing write after reporting it ([why it is needed](#why-de-needs-a-runtime-fix)) | llvm#169897 (the runtime fix becomes part of it) |
| **DE + EA** | DE + EA | all three |

Each arm's 95 % interval is compared with that of A/A (stock timed against itself, same leg):
🟢 faster (arm's lower bound above A/A's upper bound) · 🔴 slower (arm's upper bound below A/A's lower bound) ·
**=** no change (point estimate inside the A/A interval) · 🟡 unresolved (neither; the number is the point estimate) ·
† A/A wider than ±3 % ·
🟰 same machine code (EA as stock, DE + EA as DE), not timed; in the DE + runtime fix column: no race report in 5
census runs on a-p13 (FFmpeg and Redis -O0: in 50
runs on focs), so the changed runtime code never runs · — not measured · ⏳ pending · ⏸ stopped

## Results

| Program | Threads | EA, % | DE, % | DE + runtime fix, % | DE + EA, % |
|---|:-:|:-:|:-:|:-:|:-:|
| **Applications, -O0** | | | | | |
| SQLite¹ † | 8 | 🟡 +11 | 🟡 +8 | 🟡 +8 | 🟡 +10 |
| FFmpeg | 8 | = | = | 🟰 | = |
| memcached⁵ † | 8 | = | = | = | = |
| Redis | 8 | = | = | 🟰 | = |
| MySQL | 8 | = | = | = | = |
| **Applications, -O2** | | | | | |
| SQLite² † | 8 | 🟰 | = | = | 🟰 |
| FFmpeg | 8 | = | = | 🟰 | = |
| memcached † | 8 | 🟰 | = | = | 🟰 |
| Redis⁶ | 8 | 🟰 | = | = | 🟰 |
| MySQL | 8 | = | = | = | = |
| **Real applications, -O2** (fixed work: db_bench --num, compressing a fixed file, OpenSSL enc + dgst) | | | | | |
| RocksDB db_bench³ † | 8 | 🟰 | = | 🟡 +3.7 | 🟰 |
| LevelDB db_bench | 8 | 🟰 | = | 🟰 | 🟰 |
| DuckDB TPC-H | 8 | = | = | ⏳ | = |
| zstd | 8 | = | = | 🟰 | = |
| xz | 8 | 🟰 | = | 🟰 | 🟰 |
| pigz | 8 | 🟰 | = | = | 🟰 |
| brotli | 8 | 🟰 | = | 🟰 | 🟰 |
| OpenSSL | 8 | = | = | 🟰 | = |
| **Real applications, -O0** | | | | | |
| RocksDB db_bench³ † | 8 | = | = | = | = |
| LevelDB db_bench | 8 | = | = | 🟰 | = |
| DuckDB TPC-H | 8 | = | = | = | = |
| zstd | 8 | = | = | 🟰 | = |
| xz | 8 | = | 🟢 **+0.6** | 🟰 | 🟢 **+0.6** |
| pigz⁴ | 8 | 🔴 −2.3 | 🔴 −2.2 | 🔴 −2.1 | 🔴 −2.3 |
| brotli | 8 | 🟡 +1.7 | = | 🟰 | 🟢 **+1.8** |
| OpenSSL | 8 | = | = | 🟰 | = |
| **Benchmark kernels** (NPB, Rodinia, SPLASH-2x, PARSEC: small programs that model one algorithm) | | | | | |
| Rodinia hotspot † | 8 | 🟰 | 🟢 **+205** | ⏳ | 🟰 |
| NPB LU, class A † | 8 | = | 🟢 **+102** | 🟢 **+100** | 🟢 **+107** |
| NPB LU, class W † | 8 | = | 🟢 **+103** | 🟢 **+101** | 🟢 **+116** |
| NPB BT, class A † | 8 | 🟢 **+8.5** | 🟢 **+76** | ⏳ | 🟢 **+109** |
| SPLASH-2x ocean_ncp | 8 | 🟰 | 🟢 **+6** | ⏳ | 🟰 |
| SPLASH-2x water_nsquared¹¹ † | 8 | 🟰 | 🟡 +9 | ⏳ | 🟰 |
| SPLASH-2x water_spatial¹² † | 8 | 🟰 | 🟡 +10 | 🟡 +6 | 🟰 |
| Rodinia srad † | 8 | 🟰 | 🟡 +10 | 🟡 +12 | 🟰 |
| Rodinia lavaMD⁷ † | 8 | 🟰 | 🟡 +13 | 🟡 +13 | 🟰 |
| SPLASH-2x barnes † | 8 | 🟰 | = | = | 🟰 |
| PARSEC freqmine⁸ † | 8 | 🟰 | = | = | 🟰 |
| Rodinia particlefilter † | 8 | 🟰 | 🟡 +6.8 | 🟡 +5.5 | 🟡 +4.9 |
| SPLASH-2x radiosity | 8 | 🟰 | = | ⏳ | 🟰 |
| SPLASH-2x raytrace † | 8 | = | = | = | = |
| PARSEC streamcluster | 8 | 🟰 | = | ⏳ | 🟰 |
| PARSEC fluidanimate | 8 | 🟰 | 🟡 +1.8 | = | 🟰 |
| 7-Zip benchmark (llvm-test-suite; multi-threaded) | auto | = | 🟢 **+1.1** | 🟰 | 🟢 **+1.7** |
| **Chromium 156** (Intel; browser benchmarks, 95 % bootstrap over stories; 3 repetitions; static: DE + EA removes 1.4 % of the checks in the chrome binary, 3,328,362 → 3,282,284, and 1.7 % with its 552 shared libraries, 17,172,360 → 16,882,221) | | | | | |
| Speedometer 3 (20 stories)⁹ | browser | ⏳ | ⏳ | ⏳ | 🟡 +1.0 |
| blink_perf.dom (3 stories)¹⁰ | browser | ⏳ | ⏳ | ⏳ | 🟡 −0.3 |
| blink_perf.paint (2 stories) | browser | ⏳ | ⏳ | ⏳ | = |

¹ SQLite -O0, re-timed 28 Oct (4 code layouts × 4 runs, all arms in one leg): the overall score does not separate from
noise (stock against stock spans [−8.0, +5.2]; stress and trigger tests vary by up to 90 %), but single tests do:

| Test | EA | DE | DE + runtime fix | DE + EA |
|---|---|---|---|---|
| checkpoint_starvation_1 | 🟢 **+77** | 🟢 **+67** | 🟢 **+67** | 🟢 **+77** |
| walthread1 | 🟢 **+7.0** | 🟢 **+5.9** | 🟢 **+6.0** | 🟢 **+5.7** |
| walthread2 | 🟢 **+2.1** | 🟡 +0.4 | = | 🟢 **+2.5** |

² SQLite -O2: in the 22 Oct leg no single test separates from noise for DE (walthread1 +4 lies inside A/A). In the
27 Oct leg walthread1 is 🟢 +3.6 with DE and 🟡 +5.6 with DE + runtime fix (its lower bound equals A/A's upper bound;
quiet re-check queued). checkpoint_starvation_1 gains nothing at -O2.

³ RocksDB varies by ±3-5 % between runs at 8 threads. A re-time on 25 Oct with twice the runs (N = 8 per layout) did not
narrow it: -O2 A/A +2.5 [−2.2, +6.5], DE +5.5 [+1.0, +10.1]; -O0 every arm within A/A −2.3 [−4.5, +0.6]. The table
keeps the first leg.

⁴ pigz -O0, re-timed 28 Oct at N 8 (stock against stock −0.1 [−0.6, +0.8]): every arm that removes checks is **2.1-2.3 %
slower**, in both code layouts, by the same amount whatever it removes (EA, DE, DE + EA, DE + runtime fix). That points to a
shared build or layout effect rather than to the checks; the cause is not yet known.

⁵ memcached -O0 EA: three earlier legs gave =, −1.7 and −4.7; the 28 Oct re-time at N 8 gives +1.4, inside stock against
stock (±2.2 %).

⁶ Redis -O2: the composite is unchanged with DE and with DE + runtime fix, but the LRANGE workloads are slower in two
legs. 22 Oct (DE): LRANGE_600 **🔴 −2.2** [−3.4, −1.0]. 27 Oct re-run, pre-stated check (A/A ±0.5-1 %):

| Workload | DE | DE + runtime fix |
|---|---|---|
| LRANGE_300 | 🔴 −1.7 [−3.1, −0.6] | 🔴 −2.2 [−3.4, −1.3] |
| LRANGE_500 | 🟡 −1.8 [−3.3, −0.2] | 🔴 −2.4 [−3.6, −1.2] |
| LRANGE_600 | 🟡 −2.2 [−3.7, −0.6] (upper bound equals A/A's lower bound) | 🔴 −2.8 [−3.9, −1.8] |

The slowdown is small but repeatable; DE and DE + runtime fix overlap, so the fix adds no separable cost. Its cause
(code layout, or a change in which shadow cells hit) is to be profiled. At -O0, EA makes LRANGE_300/500/600 🟢 faster by
1.9-2.1 %.

⁷ lavaMD: five legs give DE +19, +6, +15 and, on 27 Oct, +13 [+2, +22] with DE + runtime fix also +13 [−2, +28]; the point
estimates agree, but stock against stock spans ±7-22 % in every setup tried (binding, passive wait, larger input), so no
leg resolves. A last attempt with 8 runs per layout is queued.

⁸ freqmine: the 25 Oct leg was unreadable (its stock-against-stock point was +13 %). Re-timed on 28 Oct (best of five
setups; stock against stock ±10 %): DE +1.9 and DE + runtime fix +2.2, both inside the noise.

⁹ Speedometer 3: the A/A of this leg is itself −2.0 [−3.4, −0.8] and excludes zero, so by the registered rule the leg
is not called, although DE + EA's interval [−0.1, +2.3] lies above it. Re-timed with 5 repetitions.

¹⁰ blink_perf.dom: one story, modify-element-classname, is 2.6-3.1 % slower with DE + EA in all 3 repetitions (A/A
−0.1 to −0.2 %); a per-story hint, checked with 8 repetitions.

¹¹ water_nsquared cannot be resolved on this host: in every setup tried on 27 Oct (as is, input ×2, 8 runs per layout)
stock against stock spans ±13-16 %, with single runs from 29 to 43 s. TSan's shadow is never reset and its memory is
constant in every run, so the spread is the program's own behaviour under TSan at 8 threads. No further leg.

¹² water_spatial: DE +10 is from the 22 Oct leg. The 28 Oct re-time (best of three setups, larger input) gives DE +5
[−14, +29] and DE + runtime fix +6 [−8, +21], but its stock-against-stock point is itself −11.5 % and the two code layouts
disagree by 30 %, so that leg is not usable either.

† The A/A interval of this row is wider than ±3 %: changes smaller than that cannot be resolved here.

**Selection.** 254 programs were screened by the share of checks DE removes, and the candidates then by the share of
executed accesses it removes. The real applications were timed regardless; kernels were timed above ~3 % of checks or
~4-5 % of accesses. Single-threaded programs (most of llvm-test-suite, the Abseil benchmarks) and 1-thread runs of the kernels were timed
too but are not reported: a race detector is not run on single-threaded programs in practice. Not timed: NPB IS/SP/FT/CG/MG/EP, GAP, the other PARSEC, SPLASH-2x
and Rodinia kernels, pbbs, LULESH, HPCCG, miniFE, XSBench. In threaded programs, spin loops make the access count vary
by 13-129 % between runs.

## Why DE needs a runtime fix

Stock TSan forgets the write that raced: after a report it clears the location's history and does not record the
access ([`DoReportRace`](https://github.com/llvm/llvm-project/blob/main/compiler-rt/lib/tsan/rtl/tsan_rtl_access.cpp)).
Later plain reads of the location are then skipped altogether, by stock as well, so removing a covered read loses
nothing; the next write is recorded again. DE removes that next write
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
(The paper's tree also merges checks and checks loop ranges; there premise P-REPORT in soundness-fixes.md accepts a
second race on a reported cell. The PR removes only plain loads and stores, so no premise is needed here.)
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
| SQLite | -O0 | 0 lost | 0 lost | 0 lost | 0 lost | 4 sites kept; walIndexRecover undetermined (stock 1/10) |
| SQLite | -O2 | 0 lost | 0 lost | 0 lost | 0 lost | 2 sites kept; walIndexRecover undetermined (stock 1/10); the two walIndexWriteHdr sites of -O0 are absent at -O2 in every build, stock included |
| memcached | -O0 | 0 lost | 0 lost | 0 lost | 0 lost | 4 sites kept at 10/10; the scheduling-dependent conn_new/conn_close family (7 sites) is undetermined (stock 2/10) |
| memcached | -O2 | 0 lost | 0 lost | 0 lost | 0 lost | all 11 sites kept: 4 at 10/10, the conn_new/conn_close family 2-3/10 in every arm, stock included |
| MySQL | -O0 | 0 lost | 0 lost | 0 lost | 0 lost | ~12,100 reports per run in every arm; 251 sites kept; 13 undetermined |
| MySQL | -O2 | 0 lost | 0 lost | 0 lost | 0 lost | ~12,000 reports per run; 226 sites kept; 26 undetermined. Closest case: dict_stats_set_persistent, stock 4/10, DE 0/10, but DE + EA, which removes every check DE removes, sees it (1/10), so DE's 0/10 is detection noise |
| FFmpeg, HEVC decoding with slice threads | -O0 | 0 lost | 0 lost | 0 lost | 0 lost | added 27 Oct (the paper workload reports no race): stock reports races in 10/10 runs; 4 sites kept at 10/10 in every arm (copy_CTB_to_hv, sao_filter_CTB in hevc_filter.c) |
| FFmpeg, HEVC decoding with slice threads | -O2 | 0 lost | 0 lost | 0 lost | 0 lost | stock 10/10 runs; 3 sites kept at 10/10 in every arm |
| FFmpeg, paper workload | -O0 | — | — | — | — | no test: 0 reports in 50 runs, stock included (the encoders are uninstrumented system libraries) |
| Redis | -O0 | — | — | — | — | no test: no build reports a race with io-threads 20 (the paper configuration, with threaded reads), 8 or 4, nor with a threaded redis-benchmark, at -O0 and -O2 |

## Open items

Everything below runs on focs (Intel) first; the AMD check follows if time allows. All of it is armed as four unattended chains on focs
(each starts when the previous ends); expected end ~31 Oct.

| Item | State |
|---|---|
| DE + runtime fix, timing: SQLite, memcached, MySQL at -O0 and -O2 (quiet), race preservation for all five (on a-p13) | ⏳ from 27 Oct, ~10-12 h |
| DE + runtime fix, every other row: added to each re-timed leg below; rows not re-timed first get a report census (5 runs each) — a row with no race report runs identical code paths and is marked 🟰, a row with reports is timed | ⏳ census on a-p13 now; timing after the re-timing |
| Resolve 🟡 and † cells, fill gaps (—): Redis -O2 LRANGE_600 first, kernels at 8 threads after an A/A pilot each (two at a time, one per half), NPB LU A EA and DE + EA, freqmine, SQLite -O0, RocksDB, the small 🟡 application cells | ⏳ after DE + runtime fix; ~29 Oct |
| Chromium 156 (Intel): proper readout of stock against DE + EA (Speedometer 3, blink_perf.dom, .paint; a possible 3 % slowdown on blink_perf.dom modify-element-classname to check), static counts, then builds for DE alone and EA alone (EA skipped if its code equals stock), DE + runtime fix by relinking, and a 5-repetition leg with all arms | ⏳ readout now; builds and leg after the re-timing |
| Race preservation for Redis and FFmpeg: workloads that run their own threaded code (Redis io-threads, FFmpeg H.264/HEVC decoding with threads); "—" stays if stock still reports no race | Redis: still no race with io-threads 20 (paper), 8 or 4 and a threaded redis-benchmark at -O0/-O2, so it stays "—". FFmpeg: HEVC decoding with slice threads at -O0 reports 47 races (libavcodec hevc_filter.c) in 2 of 2 runs; preservation with stock, DE, DE + runtime fix ⏳ on a-p13 |
| AMD (apollo): NPB BT A, LU W, hotspot, SQLite -O0 | after Intel, if time allows |

## Method

- All timings: **focs**, Intel Xeon w9-3495X (56 cores, 112 threads, max 4.8 GHz), 250 GB RAM, Linux 6.8.
- Each run is pinned to one half of the machine (CPUs 4-27, 60-83; 8-thread runs use 4-11); a run
  is discarded if the other half is busy. Every arm is timed in 2-4 code layouts and the results pooled.
- From 22 Oct on, one compiler carries all three PRs, so the arms differ only by flags. On the kernels, DE + EA and DE
  come from different legs: compare each with stock, not with each other.
- The corpus programs timed from 25 Oct (barnes, freqmine, particlefilter, radiosity, raytrace, 7-Zip) are built at -O2.
- DE and DE + EA are the PRs as they stand, without the runtime fix. DE + runtime fix compiles to DE's code; only the
  runtime differs, and only after a race report.
- Removed-check counts (screening) were taken on apollo (AMD EPYC 9115) and are not timings.
