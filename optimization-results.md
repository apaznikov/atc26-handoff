# TSan instrumentation optimizations: results

State: 12 Oct 2026. This is the short form. The full ledger with every leg, control and dead end is the previous
version of this file, commit `9126669`. Soundness, premises and audits: [`soundness-fixes.md`](soundness-fixes.md).

## Contents

| § | section | what it answers |
|---|---|---|
| 1 | [Summary](#summary) | the best figure per application, its configuration and its compile time |
| 2 | [Chromium](#chromium) | every configuration on two hosts and two story samples; what was learned |
| 3 | [Single optimizations](#single) | what each optimization adds alone, per application |
| 4 | [Further variants](#variants) | variants tried on top of the adopted forms, and the headroom left |
| 5 | [Compile time](#compile) | build time and speedup of each configuration |
| 6 | [What each optimization does](#explain) | a short text and a before/after C example for every name used here |
| 7 | [Method](#method) | how the figures were taken |

<a name="notation"></a>
**Notation** used in the tables:

| mark | meaning |
|---|---|
| <a name="kind-s"></a>S | decided at compile time: no run-time condition decides whether an access is checked (the check itself may be inlined) |
| <a name="kind-r"></a>R | a condition tested at run time decides it, with no annotation: a guard the compiler inserts, or a mode of the runtime ([DynSTC-RT](#dynstc-rt)) |
| <a name="kind-g"></a>G | uses a spec generated from the program's own assertions |
| <a name="kind-a"></a>A | uses a hand-written annotation or spec (A+R, G+R: guarded at run time as well) |
| <a name="host-f"></a>`f` | measured on the Intel host; without the mark, on the AMD host |
| <a name="leg"></a>leg, arm | a leg is one timed session in which the compared builds (its arms) run interleaved |
| <a name="aa"></a>A/A | the same binary run as two arms of one leg; its ratio shows the noise of the leg |
| N | number of repetitions (separately started legs) |
| **bold** | in section 3: a gain beyond the A/A; elsewhere: the adopted figure |
| — | not measured |
| <a name="samples"></a>first / hold-out | Chromium's two story samples: 49 stories used during development / 48 other stories fixed before they ran |
| <a name="evict"></a>eviction | TSan keeps four records per 8-byte cell of memory; a newer record can displace the one a race report needed. Stock TSan loses races this way too |

<a name="summary"></a>
## 1. Summary

Speedup = stock TSan's run time over the optimized build's; compile time is over stock TSan's build. The bold figures
are sound under the premises of [`soundness-fixes.md`](soundness-fixes.md) §6: no race that stock TSan reports is
lost except through [eviction](#evict).

| app | workload | submitted paper | adopted figure | configuration | annotations | compile time |
|---|---|---|---|---|---|---|
| SQLite | threadtest3 stress2 | 1.71× | **1.19×** | [LO-OBJ-G](#lo-obj-g) | none: the spec is generated from the program's assertions | +30 % |
| FFmpeg | four transcodes of one film | 1.57× | **1.29×** | [DynSTC-RT](#dynstc-rt) + [N1](#n1) + [N1-ST](#n1-st) | none | +97 % |
| Redis | 7 data-heavy commands, 8 I/O threads | 1.45× | **1.54×** | [FE-INL](#fe-inl) + [N1](#n1) + [quiet threads](#quiet) | none | +168 % |
| MySQL | Release, sysbench insert/update/delete, 24 connections | 1.16× | **1.13×** | [FE-INL](#fe-inl) | none | +16 % |
| memcached | pipelined 32-key gets, 190-byte keys | 1.07× | **1.80×** | [EVCONF-CHECKED](#evconf-checked) | none | not measured |
| Chromium | blink_perf, [hold-out](#samples) sample, Intel host | 1.39× | **1.25×** | [CR1+N1](#cr1) | none | +10 % |

Other measured configurations of the same applications:

| app | configuration | speedup | compile time | note |
|---|---|---|---|---|
| SQLite | [the paper's analyses](#paper) with the soundness fixes | ≈ 1.0× | | |
| SQLite | [LO-OBJ-G](#lo-obj-g), hand-written spec | 1.25× | | |
| SQLite | LO-OBJ-G with parser parameters derived and owner checks at run time | 1.24× | | another compiler root; audit and preservation run owed |
| SQLite | [LO-MARK](#lo-mark) | 1.06-1.26× | | three [legs](#leg); not separated from the generated spec (1.23× in the best leg); not a claim |
| FFmpeg | [N1 by static rule s4](#n1-place) instead of N1 everywhere | 1.23× | +36 % | one repetition per code offset |
| Redis | [FE-INL](#fe-inl) + [N1](#n1), no quiet threads | 1.10× | | static only |
| memcached | analyses + [EA-CONTENTS](#ea-contents) + [SWMR-ROOTS](#swmr-roots) + [FE-INL](#fe-inl) + [MEMINTR](#memintr) | 1.02× | +11 % | static only |
| memcached | [EVCONF](#evconf), 7 annotated fields | 1.81× | +35 % | |
| Chromium | [CR1](#cr1) | 1.10× | +1 % | static only |
| Chromium | [S56](#n1-place) | 1.16× | not sampled | |

- Hosts: AMD = 2 × EPYC 9115 (every row except Chromium); Intel = Xeon w9-3495X (Chromium; its AMD figures are in
  [section 2](#chromium)).
- SQLite's figure is completed statements per 60 s of a lock-contended mix dominated by integrity checks (33-36 %) and
  schema statements; the write loops are 7-9 % of it. It is a throughput ratio, not a speedup of write work.
- SQLite without [dominance elimination](#de) and [peeling](#peel) runs at the same speed (1.185 against 1.187, one
  leg, the earlier generated spec) and builds at +13 % instead of +25 % (12 Oct, same method as [section 5](#compile)).
- Redis: a fresh build of the same configuration read 1.49× on 11 Oct; that build is the one timed in
  [section 5](#compile).
- FFmpeg: of the 1.29×, 1.085× is the runtime option of [DynSTC-RT](#dynstc-rt) alone; the instrumentation gives 1.18×.
- memcached's 1.80× and MySQL's 1.13× are over upstream LLVM's TSan; ours and upstream's stock differ by -4 to +1.5 %.
- [The paper's analyses](#paper), with the soundness fixes, read about 1.0× on SQLite, memcached and Chromium on the
  camera-ready compiler; the adopted figures are the analyses with their extensions ([section 6](#explain)).
- Still open beside the figures: five premises await a decision (`soundness-fixes.md` §6b; two of them are EVCONF's);
  LO-MARK's premises are not written up there; two unexplained server deaths in 104 MySQL runs under the inline hit
  test (none in 319 runs without it, none in 80 later runs).

<a name="chromium"></a>
## 2. Chromium

Chromium 127.0.6533.119, component build (486 libraries) with Chromium's debug assertions (DCHECKs) on, compiler
`tsan-cr-8345a0396fa5` (`tsan-cr2-429bd8cf76b9` for S56). [First sample](#samples): 49 Blink stories of six suites,
used during development. [Hold-out sample](#samples): 48 other stories, fixed before they ran. Speedup over stock
TSan, N=3 unless noted. The configurations are defined [here](#cr1).

| configuration | Intel, [first](#samples) | Intel, [hold-out](#samples) | AMD, first | AMD, hold-out |
|---|---|---|---|---|
| [Sound](#cr1) ([the four analyses](#paper)) | 1.005 | — | 0.995 | — |
| Sound + [DE](#de), no peeling | 1.012 | — | 0.999 | — |
| [AllOpt](#cr1) (Sound + [DE](#de) + [peeling](#peel)) | 0.993 | — | 1.011 | — |
| [CR1](#cr1) | 1.104 | 1.101 | 1.068 | 1.057 |
| [CR1](#cr1) without the [library-facts table](#cr1) | 1.104 | 1.106 (N=2) | 1.070 | — |
| [S56](#n1-place) | 1.186 | 1.157 | 1.086 | 1.054 |
| [CR1+N1](#cr1) | 1.286 | 1.248 | 1.081 | 1.017 |

What each step adds is in [section 3](#single-chromium). Read with two limits: this is not the build Chromium's own
TSan bots use (static, no DCHECKs; static builds were not measured, and the difference favours inlined paths); and a
difference under about 1.5 % is within what code placement alone does (a layout twin of [CR1](#cr1) reads 1.013 over
it).

- **On Chromium the gain comes from the form of the instrumentation.** [The paper's analyses](#paper) read within
  about 1 % of stock on both hosts; [FE-INL](#fe-inl) (with [MEMINTR](#memintr)) in [CR1](#cr1) and the inline hit
  test [N1](#n1) carry the gain. [SWMR-ROOTS](#swmr-roots) is switched on in CR1 but does nothing in a build
  compiled unit by unit (`soundness-fixes.md` §6e).
- **[CR1](#cr1) holds on unseen stories**: 1.10× on Intel, 1.06× on AMD.
- **The [inline hit test](#n1) pays only on Intel**: +13 to +17 % over CR1. On AMD's hold-out it is 4 % below CR1
  (four readings of the same sign), and [S56](#n1-place) equals CR1 there.
- **Why the hosts differ is not established.** Excluded: thread count (Intel pinned to 16 threads reads the same),
  the container, fonts and paths (AMD on the host reproduces the container within 0.006). Counters show the hit test
  removing cycles in the runtime and adding about as many outside it on both hosts.
- **The [library-facts table](#cr1) is not needed**: CR1 without it reads the same everywhere.
- **Whole `paint` suite** (AMD, N=2): [CR1](#cr1) 1.062, [CR1+N1](#cr1) 1.160, [S56](#n1-place) 1.169; without its
  multi-column layout stories 1.048, 1.030, 1.044.
- **Code and memory**: code 1.18× stock's for CR1 and 3.0× for CR1+N1. Largest process's peak RSS: CR1 1.09×, S56
  1.27×, CR1+N1 1.77× (mapped code; allocated memory unchanged).
- **compiler-rt TSan tests with the flags** (391 tests run): [CR1](#cr1), [CR1+N1](#cr1) and [S56](#n1-place) pass
  all. The [hit-test](#n1) flag alone fails one test at -O0 (a link error from an unused declaration; fix decided
  12 Oct, not yet in a frozen compiler).
- **Open**: one CR1+N1 Speedometer story failed in 19 runs (a renderer disappeared, no sanitizer output; it did not
  recur).

**Race preservation.** Runs that report the race, over all sessions (10 runs of the Blink stories and at least 2 of
Speedometer per build in the main table):

| race (where it shows) | stock | [CR1](#cr1) | [CR1+N1](#cr1) | [S56](#n1-place) |
|---|---|---|---|---|
| crashpad `WorkerThread::Stop` (Speedometer) | 27 of 27, stock and CR1 together | | 10 of 19 | 3 of 4 |
| perfetto start-up (`paint`) | 0 of 30 (4 of 9 with the shadow-reset option) | 4 of 20 | 3 of 10 | — |
| PartitionAlloc bit-field (Speedometer) | 7 of 15 | 6 of 12 | 2 of 19 | 2 of 4 |

Every race that stock reports is reported by every build in at least one run. The crashpad runs without a report are
[eviction](#evict) in the unmodified runtime (`soundness-fixes.md` §6e); the lower PartitionAlloc rate under CR1+N1 is
not explained.

<a name="single"></a>
## 3. Single optimizations

Change of speed in percent from adding one optimization. Rows down to [MEMINTR](#memintr): over
[the paper's analyses](#paper) with every soundness fix. Rows below: on top of their application's configuration.

Marks S, R, G, A, `f`, bold and "—" are explained under [Notation](#notation).

| optimization | kind | SQLite | memcached | Redis | MySQL | FFmpeg |
|---|---|---|---|---|---|---|
| [DynSTC-RT](#dynstc-rt) | [R](#kind-r) | +0.4 | −0.3 | [`f`](#host-f) −2.2 | +0.1 | [`f`](#host-f) **+12.0** |
| [DynSTC](#dynstc) (the paper's guard) | [R](#kind-r) | — | −0.5 | −5.5 | −0.7 | [`f`](#host-f) **+10.1** |
| [N1](#n1) | [S](#kind-s) | −0.8 | +0.3 (−5 on memcached's adopted configuration) | [`f`](#host-f) +2.5 | −1.7 | [`f`](#host-f) **+5.0** |
| [N1-ST](#n1-st) (over N1 + DynSTC-RT) | [R](#kind-r) | +0.2 | −0.2 | [`f`](#host-f) −1.4 | −1.8 | [`f`](#host-f) **+16.9** |
| [N1-L](#n1-place) | [S](#kind-s) | +0.3 | −0.9 | [`f`](#host-f) −2.5 | −0.1 | [`f`](#host-f) **+3.0** |
| [FE-INL](#fe-inl) | [S](#kind-s) | −0.7 | +0.8 | [`f`](#host-f) **+4.8** | **+3.6** | [`f`](#host-f) 0.0 |
| [FE-INL + VWIDE-loops](#vwide) | [R](#kind-r) | +0.1 | +0.6 | [`f`](#host-f) **+7.2** | **+4.6** | [`f`](#host-f) +1.8 |
| [FE-SINK](#fe-sink) (nothing on top of FE-INL) | [S](#kind-s) | +1.1 | −1.7 | [`f`](#host-f) **+4.1** | **+2.8** | [`f`](#host-f) −1.2 |
| [MEMINTR](#memintr) | [S](#kind-s) | +2.5 | +0.9 | [`f`](#host-f) −2.5 | +0.3 | [`f`](#host-f) 0.0 |
| [EA-CONTENTS](#ea-contents) | [S](#kind-s) | — | **+1.4** | — | — | — |
| [SWMR-ROOTS](#swmr-roots) | [S](#kind-s) | — | **+0.9** | — | — | — |
| [EVCONF](#evconf) (with the two rows above) | [A](#kind-a)+[R](#kind-r) | — | **+31.1** | — | — | — |
| [EVCONF-RANGES](#evconf) | [A](#kind-a)+[R](#kind-r) | — | **+18.5** | — | — | — |
| [EVCONF-ARGS](#evconf) | [A](#kind-a)+[R](#kind-r) | — | **+7.3** | — | — | — |
| [EVCONF-INTERCEPT](#evconf) | [A](#kind-a)+[R](#kind-r) | — | **+7.5** | — | — | — |
| [EVCONF-CHECKED](#evconf-checked) (over upstream stock) | [R](#kind-r) | — | **+80** | — | — | — |
| [Quiet threads](#quiet) | [R](#kind-r) | — | — | **+31.9** | — | — |
| [LO-OBJ-G](#lo-obj-g), generated spec, earlier generator (over stock) | [G](#kind-g)+[R](#kind-r) | **+15.7** | — | — | — | — |
| [LO-OBJ-G](#lo-obj-g), hand-written spec (over stock) | [A](#kind-a)+[R](#kind-r) | **+25.0** | — | — | — | — |

<a name="single-chromium"></a>
**Chromium**, step by step: change in percent over the configuration named in "over", from the table of
[section 2](#chromium), both arms taken in the same [legs](#leg). Steps under 1.5 % are inside the code-placement
bound.

| step | over | Intel, [first](#samples) | Intel, [hold-out](#samples) | AMD, first | AMD, hold-out |
|---|---|---|---|---|---|
| [The four analyses](#paper) ([Sound](#cr1)) | stock | +0.5 | — | −0.5 | — |
| [DE](#de) | [Sound](#cr1) | +0.7 | — | +0.5 | — |
| [Peeling](#peel) | Sound + [DE](#de) | −1.9 | — | +1.1 | — |
| [FE-INL](#fe-inl) + [MEMINTR](#memintr) (= [CR1](#cr1); [SWMR-ROOTS](#swmr-roots) is on but inert here) | [Sound](#cr1) | **+9.8** | — | **+7.4** | — |
| [library-facts table](#cr1) | [CR1](#cr1) without it | −0.1 | −0.3 | −0.2 | — |
| [N1](#n1) at every check | [CR1](#cr1) | **+16.5** | **+13.4** | +1.2 | −3.8 |
| [N1 in small functions, S56](#n1-place) | [CR1](#cr1) | **+7.4** | **+5.1** | +1.7 | −0.3 |
| [N1 in small loops, K4](#n1-place) (indicative: one repetition, CR1 from other legs) | [CR1](#cr1) | +0.2 | — | −0.4 | — |

<a name="variants"></a>
## 4. Further variants

The variants below were tried on top of the adopted forms, either built and timed or bounded by a count of executed
checks, and add little on these workloads; they are recorded so that they are not tried again. One row per family;
shares are of executed checks. Details: commit `9126669`, tables 3 and 6.

| family | variants tried | added on top of the adopted form |
|---|---|---|
| [N1](#n1) variants | compact or batched tests; shared thread-state load; test for relaxed atomics; out-of-line entries saving fewer registers | ±1 %, or −2 to −4 %; the batched form is not sound |
| [N1](#n1) placement for speed | profile hot list; static frequency; loops only | none beats the full version in speed; they trade speed for compile time ([section 5](#compile)) |
| [FE-INL](#fe-inl) variants | lazy entry; only hot functions; one exit; no entry/exit in functions without accesses | ±1 %, MySQL −5 % for one exit; the last changes report stacks and was not taken |
| [N1](#n1), [FE-INL](#fe-inl), [MEMINTR](#memintr) on top of SQLite's [LO-OBJ-G](#lo-obj-g) | each added alone | [FE-INL](#fe-inl) +4.5 %, [MEMINTR](#memintr) +3.5 % inside a wide [A/A](#aa); [N1](#n1) −18.5 % |
| More optimization before instrumentation | LTO; Attributor; extra passes | nothing; the Attributor miscompiles |
| Loop guard (a relative of [peeling](#peel)) | a loop-invariant access checked once per sync-free stretch | Redis +2 to +3 %, the others within ±1 % (MySQL down to −4 % in one form); not taken |
| [Dominance elimination](#de), further rules | covered checks deleted instead of [verified](#de); one range check per loop; adjacent fields merged; a cover on every path; covers across loop iterations; post-dominance with proven termination; stronger alias analysis | within ±1 % on top of the adopted [DE](#de) (Redis −3.8 % for deletion); each rule reaches at most 2 % of checks beyond what [DE](#de) already covers. An earlier package of the merging and loop-range rules read memcached +3.6 %, MySQL +3 to +6 %, Redis −2.6 % and was superseded |
| [Escape analysis](#ea), finer | whole-program summaries; per-call-site parameter facts; slots and field chains; fresh return values; thread-local roots | each reaches under 2-3 % of checks beyond the adopted [EA](#ea) and [EA-CONTENTS](#ea-contents); what remains on SQLite is mostly heap memory reachable from shared structures (34-42 % on memcached and Redis, 7.5 % on FFmpeg), part of which the [run-time ownership checks](#own) take |
| [STC](#stc), [SWMR](#swmr), [LO](#lo), finer | closed-world facts; join-aware STC; thread roles; publish-once; written-before-publication; POSIX calls transparent to LO | each reaches under 3 % of checks beyond the adopted [STC](#stc), [SWMR](#swmr), [LO](#lo) and [SWMR-ROOTS](#swmr-roots); two variants were not sound and were withdrawn |

Headroom left to the static analyses (share of executed checks that a stronger proof could still remove). On SQLite,
memcached and Redis part of it is taken at run time by the [ownership checks](#own):

| not yet proven statically | SQLite | memcached | Redis | MySQL | FFmpeg |
|---|---|---|---|---|---|
| [DE](#de): same address twice, no acquire between, both kept | 3.4 % | 0.04 % | 3.4 % | 5.5 % | 5.9 % |
| [DE](#de): neither check dominates the other | 4.4 % | 0.08 % | 5.8 % | 1.0 % | 8.6 % |
| [EA](#ea): memory that is in fact one-thread or consistently ordered | 72 % | 55 % | 63 % | 49 % | 70 % |
| share of the row above that is heap reachable from shared structures | 79 % | 34 % | 42 % | — | 7.5 % |

**A further application screened: zstd** (12 Oct; zstd 1.5.7, Silesia corpus, 16 workers; untimed shared host, so
indications only, 5 rounds). Stock TSan reports no race. Over stock, compressing at level 19: the analyses with
[DE](#de) 1.04×, with [FE-INL](#fe-inl) 1.03×, with [N1](#n1) 1.08× (stock's own spread 1.04); at level 3 and with
`--long` nothing leaves the spread; decompression is single-threaded. [DE](#de) removes 4.4 % of the static checks
and [EA](#ea) 2.0 %; [STC](#stc), [SWMR](#swmr) and [LO](#lo) find nothing in it. zstd reads mostly through
unaligned-access helpers, which [N1](#n1) does not cover (8,131 call sites). Build time with N1 +86 %, code 2.5×.

<a name="compile"></a>
## 5. Compile time

Fresh trees, no ccache, 8 jobs (MySQL 6), stock and the optimized build alternating, medians of 3, wall seconds.
For Redis and memcached the optimized build includes the summary step: a pass over the whole program that
[quiet threads](#quiet) and the cross-unit analyses need before the units are compiled.

| app | configuration | speedup | stock build | optimized build | compile-time overhead |
|---|---|---|---|---|---|
| memcached | no annotations (analyses + [EA-CONTENTS](#ea-contents) + [SWMR-ROOTS](#swmr-roots) + [FE-INL](#fe-inl) + [MEMINTR](#memintr)) | 1.02× | 7.25 s | 8.03 s | +11 % |
| memcached | [EVCONF](#evconf), 7 annotated fields ([EVCONF-CHECKED](#evconf-checked), 1.80×, was not timed) | 1.81× | 6.47 s | 8.71 s | +35 % |
| Redis | [FE-INL](#fe-inl) + [N1](#n1) + [quiet threads](#quiet) | 1.49× | 5.37 s | 14.38 s | **+168 %** |
| SQLite | [LO-OBJ-G](#lo-obj-g), generated spec | 1.19× | 33.27 s | 43.28 s | +30 % |
| FFmpeg | [N1](#n1) at every check | 1.285× | 76.10 s | 149.88 s | +97 % |
| FFmpeg | [N1 by static rule s1](#n1-place) | 1.241× | 76.10 s | 110.77 s | +46 % |
| FFmpeg | [N1 by static rule s4](#n1-place) | 1.227× | 76.10 s | 103.53 s | +36 % |
| MySQL | Release, [FE-INL](#fe-inl) | 1.13× | 874.75 s | 1015.13 s | +16 % |
| Chromium | [CR1](#cr1) (400 sampled units, CPU time) | 1.10× ([`f`](#host-f)) | — | — | +1.1 % (1.004-1.018) |
| Chromium | [CR1+N1](#cr1) (same sample) | 1.25× ([`f`](#host-f)) | — | — | +10.4 % (1.076-1.142) |

- Redis's 1.49× is the build that was timed here; the leg behind section 1 read 1.54×.
- SQLite: generating the spec takes 52.9 s more, once per program version. The earlier generator's spec (1.17× in the
  same leg) builds at +28 %.
- A rerun of 11 Oct on later compilers reproduced the rows within about ten points, except memcached's first row
  (+24 % against +11 %; in a program this small the difference is 1.4 s) and FFmpeg's s4 (+28 %).

- **The cost is the [inline hit test](#n1)**: 133-140 bytes and two or three extra blocks per check. Redis by CPU
  time over stock: analyses +1.6 %; with [elimination](#de) and [peeling](#peel) +17 %; with
  [inlined entry/exit](#fe-inl) +24 %; with the hit test +157 %. The pass itself is 1 % of the added time; the rest
  is the back end working over the larger code.
- **No change of the pass lowers it**: prototypes of a smaller test gave at most 1-2 % (an assembly form −10 % with
  1.5-1.9 % more code). The one design that would is a back-end pseudo-instruction, recorded as an idea in
  [`runtime/ideas.md`](runtime/ideas.md) and not built.
- **Fewer tests cost speed on Redis** (AMD, over stock; overhead as CPU time of the units alone, on which scale the
  adopted build is +174 %): adopted 1.49×; [static rules](#n1-place) [s1](#n1-place) 1.34× (+73 %),
  [k2](#n1-place) 1.32× (+46 %), [s4](#n1-place) 1.29× (+42 %); no [hit test](#n1) 1.28× (+26 %). By the table's
  method (wall, summary step included) the points are 1.49× at +168 %, s4 at +121 % and no hit test at +113 %.
- **FFmpeg**: no static rule separates the hot checks from the bulk ([s1](#n1-place) keeps 48 % of the hot locations for 23 % of
  the code growth; function-size caps keep 2-13 %). The whole speed difference is in the stream-copy transcode.
- **SQLite**, earlier generated spec, wall time by the table's method (12 Oct; stock 34.31 s): with all flags +25 %;
  without [peeling](#peel) +16 %; without [elimination](#de) and peeling +13 %, at the same speed. By CPU time of the
  units alone: +37 %, +27 %, +20 %; the owner checks of [LO-OBJ-G](#lo-obj-g) and the analyses are what remains.
- **Redis's summary step** was 7.9 s of 17.8 s (+230 %). Not rerunning `make -n` and the dependency build, and
  running the whole-program analyses in parallel, brings it to 4.6 s (adopted 12 Oct). Summaries and server code are
  identical.

<a name="explain"></a>
## 6. What each optimization does

The examples are schematic C. `__tsan_read4(p)` / `__tsan_write4(p)` stand for TSan's check of a 4-byte access; real
entry names and the inlined code differ in detail. Each entry: what it does, then, where it helps, the code before
and after.

| group | entries |
|---|---|
| [The paper's analyses](#paper) | [STC](#stc) · [SWMR](#swmr) · [LO](#lo) · [EA](#ea) · [DE](#de) · [Peeling](#peel) · [DynSTC](#dynstc) |
| [The form of the instrumentation](#form) | [FE-INL](#fe-inl) · [FE-SINK](#fe-sink) · [N1](#n1) · [N1 placement: N1-L, K4, s1, s4, k2, S56](#n1-place) · [DynSTC-RT](#dynstc-rt) · [N1-ST](#n1-st) · [VWIDE-loops](#vwide) · [MEMINTR](#memintr) |
| [Refinements of the analyses](#refine) | [EA-CONTENTS](#ea-contents) · [SWMR-ROOTS](#swmr-roots) |
| [Ownership checked at run time](#own) | [EVCONF](#evconf) · [EVCONF-CHECKED](#evconf-checked) · [Quiet threads](#quiet) · [LO-OBJ-G](#lo-obj-g) · [LO-MARK](#lo-mark) |
| [Chromium configurations](#cr1) | [Sound · AllOpt · CR1 · CR1+N1 · S56 · library-facts table](#cr1) |

<a name="paper"></a>
### The paper's analyses

---

<a name="stc"></a>
#### STC, single-threaded context

Code that runs only before the program creates its first thread needs no checks.

```c
int main(void) {
    g_config.n = 4;                 // before: __tsan_write4(&g_config.n);  after: nothing
    pthread_create(&t, 0, worker, 0);
}
```

---

<a name="swmr"></a>
#### SWMR, single writer / multiple readers

If every write to a variable happens in single-threaded mode, its reads
need no checks: two reads cannot race.

```c
void *worker(void *a) {
    int n = g_config.n;             // before: __tsan_read4(&g_config.n);   after: nothing
}                                   // (g_config.n is written only in main, before the threads)
```

---

<a name="lo"></a>
#### LO, lock ownership

If every access to a variable that can run in multi-threaded mode holds one common lock, none
of them is checked.

```c
pthread_mutex_lock(&lock);
g_counter++;                        // before: __tsan_write4(&g_counter);   after: nothing
pthread_mutex_unlock(&lock);        // (every other access to g_counter also holds `lock`)
```

---

<a name="ea"></a>
#### EA, escape analysis

An object whose address never leaves its thread is not checked.

```c
void handle(void) {
    req_t req;                      // never stored to a global, a shared object or another thread
    req.id = next_id();             // before: __tsan_write4(&req.id);      after: nothing
}
```

---

<a name="de"></a>
#### DE, dominance elimination

A second access to the same address needs no new check if a checked access dominates
it and nothing between them can synchronize. In the adopted builds the covered check is not deleted but *verified*:
it becomes a short inline test that the earlier record is still in the shadow cell, and calls the runtime only if not.

```c
x = p->len;                         // __tsan_read4(&p->len)
y = p->len + 1;                     // before: __tsan_read4(&p->len)
                                    // after:  if (record_of_the_check_above_is_gone(&p->len)) __tsan_read4(&p->len);
```

---

<a name="peel"></a>
#### Peeling

The first iteration of a loop is split off so that its check dominates, and so covers, the same access in
the remaining iterations.

```c
for (i = 0; i < n; i++) s += *p;    // before: __tsan_read4(p) in every iteration
// after:
if (n > 0) { __tsan_read4(p); s += *p; }
for (i = 1; i < n; i++) s += *p;    // verified against the peeled check, as in DE
```

---

<a name="dynstc"></a>
#### DynSTC

A guard compiled around the checks skips them while the program has one thread.

```c
if (__tsan_multithreaded) __tsan_read4(p);   // before: __tsan_read4(p) unconditionally
x = *p;
```

<a name="form"></a>
### The form of the instrumentation

---

<a name="fe-inl"></a>
#### FE-INL, inlined function entry and exit

TSan keeps a shadow call stack for its reports and calls the runtime at
every function entry and exit. FE-INL does the push and the pop in place.

```c
int get(obj *o) {
    __tsan_func_entry(pc);          // after: *thr->shadow_stack_pos++ = pc;   (a few instructions)
    ...
    __tsan_func_exit();             // after: thr->shadow_stack_pos--;
}
```

---

<a name="fe-sink"></a>
#### FE-SINK

The entry is moved down to the first point that needs the frame, so early returns record nothing.

```c
int get(obj *o) {
    if (!o) return 0;               // after: no entry/exit on this path
    __tsan_func_entry(pc); ...
}
```

---

<a name="n1"></a>
#### N1, inline hit test

Most checks find that the same thread already recorded the same access in the shadow cell
and return. N1 does that comparison in place and calls the runtime only on a miss. It costs code: 133-140 bytes per
check.

```c
x = p->field;
// before:  __tsan_read4(&p->field);
// after:   if (none of the cell's four records == this access by this thread)
//              __tsan_read4_miss(&p->field);
```

---

<a name="n1-place"></a>
#### Where the hit test is placed (N1-L, K4, s1, s4, k2, S56)

The same test at fewer checks, to save code and compile
time; everywhere else the plain runtime call stays.

| name | the test is inlined only |
|---|---|
| N1-L, K4 | in loops with at most 20 checked accesses |
| s1, s4 | at checks whose block runs at least 1× / 4× per function entry by static frequency |
| k2 | at the 2 statically hottest checks of each function |
| S56 | in functions of at most 56 IR instructions (small accessors) |

---

<a name="dynstc-rt"></a>
#### DynSTC-RT

[DynSTC](#dynstc)'s idea inside the runtime instead of the compiled code: while one thread is alive the runtime
records nothing. It is a runtime option; the binary is unchanged.

---

<a name="n1-st"></a>
#### N1-ST

With [DynSTC-RT](#dynstc-rt) on, the inline test of [N1](#n1) itself is skipped while the program has one thread.

```c
// N1:     if (no_matching_record(p)) __tsan_read4_miss(p);
// N1-ST:  if (multithreaded && no_matching_record(p)) __tsan_read4_miss(p);
```

---

<a name="vwide"></a>
#### VWIDE-loops

[DE](#de)'s verified form extended to checks inside loops that no earlier check dominates: the check
becomes the inline hit test of [N1](#n1) where an earlier check of the same address in the same epoch is likely (a
static estimate), and only in loop blocks. An unconditional access in a loop needs none of this:
[peeling](#peel) and DE cover it. It differs from [N1-L](#n1-place), which inlines the test at every check of a small
loop: here the test goes only where an earlier check of the same address is likely. It is marked R because it comes
with the verified mode of DE; the test itself is N1's.

```c
for (i = 0; i < n; i++)
    if (v[i] > 0)
        sum += a->total;            // conditional: no iteration's check dominates the next
                                    // before: __tsan_read4(&a->total) each time
                                    // after:  inline hit test, runtime call only on a miss
```

---

<a name="memintr"></a>
#### MEMINTR, one-sided memory intrinsics

A `memcpy` whose source is a constant or a private object checks only its
destination.

```c
char buf[64];                       // private to this thread
memcpy(shared->data, buf, 64);      // before: both ranges checked;  after: only shared->data
```

<a name="refine"></a>
### Refinements of the analyses

---

<a name="ea-contents"></a>
#### EA-CONTENTS

Reading a pointer out of a private container and publishing that pointer no longer makes the
container itself shared.

```c
item *slots[8];                     // private array
publish(slots[i]);                  // the item escapes; the array does not
slots[j] = 0;                       // before: __tsan_write8(&slots[j]);  after: nothing
```

---

<a name="swmr-roots"></a>
#### SWMR-ROOTS (SWMR thread roots)

SWMR with thread entry functions as roots: reads of a global need no checks if
every write to it precedes the creation of each thread that can reach it, even if other threads already exist.

```c
settings.port = 11211;              // every write is before the create
pthread_create(&t, 0, worker, 0);   // worker and what it calls are the only readers
// in worker:  use(settings.port);  // before: __tsan_read4(&settings.port);  after: nothing
```

<a name="own"></a>
### Ownership checked at run time

---

<a name="evconf"></a>
#### EVCONF, event-loop confinement

In an event-driven server the buffers and response objects of a connection are
used by one worker thread. Seven pointer fields named in a spec (the connection's read buffer and response pointers
among them) mark that memory; accesses to the memory they point to run unchecked while a run-time guard holds, and
with the normal checks otherwise. The connection's own fields stay checked. The guard is the spec's condition read at
run time: a setting that is zero while no other thread may touch connections, and a latch set when a connection is
lent to another thread. EVCONF-RANGES puts the same guard around `memset`/`memcpy` of that memory, EVCONF-ARGS clones
a callee with the covered pointer argument, EVCONF-INTERCEPT covers `memchr`, `strlen`, `bcmp` and `memcmp` on it.

```c
char *p = c->rcurr;                 // c->rcurr itself: checked as before
n = p[0];                           // before: __tsan_read1(p)
                                    // after:  if (settings.idle_timeout != 0 || evconf_latch) __tsan_read1(p);
memset(c->resp, 0, sz);             // the same guard around the range check (EVCONF-RANGES)
```

---

<a name="evconf-checked"></a>
#### EVCONF-CHECKED

The same elision with no annotation: the analysis proposes the ownership candidates, and run-time
owner checks plus a marker in the shadow make each elision sound. A failed check voids the run instead of losing a
race silently.

---

<a name="quiet"></a>
#### Quiet threads

Redis's main thread does nearly all the work while the I/O and background threads sleep. The
compiler derives from the whole program where the other threads are parked; while every one of them has been quiet
since a lock release that the main thread acquired, the main thread skips its checks, range checks included.

```c
// main thread, per command:
if (all_other_threads_quiet()) {    // guard
    reply->len += n;                // unchecked
    memcpy(reply->buf, s, n);       // unchecked
} else { /* the same code with the checks */ }
```

---

<a name="lo-obj-g"></a>
#### LO-OBJ-G, objects owned by a lock

LO per object instead of per variable: a spec says which lock protects which
object, and accesses to the object are unchecked while the thread holds that lock (tested at run time). The spec is
generated from the program's own assertions such as `assert(sqlite3_mutex_held(pBt->mutex))`, or written by hand.

```c
sqlite3_mutex_enter(pBt->mutex);
pBt->nPage++;                       // before: __tsan_write4(&pBt->nPage)
                                    // after:  if (!held(pBt->mutex)) __tsan_write4(&pBt->nPage);
sqlite3_mutex_leave(pBt->mutex);
```

---

<a name="lo-mark"></a>
#### LO-MARK

[LO-OBJ-G](#lo-obj-g) without assertions or hand annotations; the object's lock is still known per type (where the
object's owner pointer is and which field of the owner is the lock). A guard before the object's accesses tests
whether the object's first shadow cell holds the marker of the lock this thread holds now; if not, the runtime checks
that the thread holds the owner's lock and marks the object, else the accesses stay checked. The marker stays after
unlock. A checked access by a thread that does not hold the lock and meets a marker voids the run, so an unprotected
access is never lost silently. Measured in three legs (1.06× on Intel, 1.14× and 1.26× on AMD); not separated from
[LO-OBJ-G](#lo-obj-g)'s generated spec; its premises are not decided; not adopted.

```c
sqlite3_mutex_enter(pBt->mutex);
pBt->nPage++;                       // before: __tsan_write4(&pBt->nPage)
// after: if (shadow_marker(pBt) != my_lock_marker && !__tsan_lo_mark(pBt, owner, &desc))
//            __tsan_write4(&pBt->nPage);
```

---

<a name="cr1"></a>
### Chromium configurations

| name | what is switched on |
|---|---|
| Sound | [STC](#stc), [SWMR](#swmr), [LO](#lo), [EA](#ea) |
| AllOpt | Sound + [DE](#de) + [peeling](#peel) |
| CR1 | Sound + [SWMR-ROOTS](#swmr-roots) (inert in Chromium's unit-by-unit build) + [FE-INL](#fe-inl) + [MEMINTR](#memintr) |
| CR1+N1 | CR1 + [N1](#n1) at every check |
| S56 | CR1 + [N1](#n1-place) only in functions of at most 56 IR instructions |
| library-facts table | a list of facts about library functions for the analyses; CR1 reads the same without it |

<a name="method"></a>
## 7. Method

- Applications: stock and the optimized line are built by the same compiler and run in the same leg, alternating;
  the figure is the geometric mean over 4 code offsets. Each leg carries an [A/A](#aa) arm (the base binary twice).
- Chromium: one build per configuration, no code offsets; arm order rotated between repetitions; intervals are
  bootstrap over stories; TSan reporting off in timed runs.
- Race preservation: 10 stock against 10 optimized runs per app, no race-report site lost (memcached 4 sites kept,
  SQLite 3, MySQL 232-236, Redis and FFmpeg seeded races, since their stock runs report none).
- Compilers: `tsan-cr-8345a0396fa5` (camera-ready root), `tsan-cr2-429bd8cf76b9` (the same plus the static placement
  rules), `tsan-n1s-e52d8787b59e` (the same line with Redis's phase derivation). Three figures of section 1 come
  from other roots: memcached's 1.80× (`tsan-ecc-b54ce0196884`), SQLite's 1.24× (`tsan-carve-da51c2dd8b3c`) and
  LO-MARK (`tsan-lomark-718c7520052c`).
- Premises used by the adopted figures: `soundness-fixes.md` §6.
