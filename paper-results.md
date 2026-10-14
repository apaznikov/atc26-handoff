# Results for the paper

State: 12 Oct 2026. Only figures meant for the camera-ready text, in the order of the paper's evaluation. Every
entry is marked **final** (taken by [`measurement-protocol.md`](measurement-protocol.md)) or **current** (the best
figure so far, to be replaced by the final record leg). The full ledger is
[`optimization-results.md`](optimization-results.md); the mapping of the new optimizations onto the paper's
configurations is `camera-ready-mapping.md` (branch `camera-ready-mapping`).

Hosts: AMD = 2 × EPYC 9115; Intel = Xeon w9-3495X. Workloads are the current ones, which differ from the
submission's (they were chosen for a larger share of TSan overhead); the text must say so.

## 1. Setup

| app | workload | host | compiler root | build |
|---|---|---|---|---|
| SQLite | threadtest3, stress2 (60 s) | AMD | `tsan-lomark-913cae99c263` | |
| FFmpeg | four transcodes of one film | AMD | `tsan-cr2-429bd8cf76b9` | |
| Redis | 7 data-heavy commands, 8 I/O threads | AMD | `tsan-n1s-e52d8787b59e` | |
| MySQL | sysbench insert/update/delete, 24 connections | AMD | `tsan-cr-8345a0396fa5` | `-O2 -g` |
| memcached | pipelined 32-key gets, 190-byte keys | AMD | `tsan-ecc-b54ce0196884` | |
| Chromium 127.0.6533.119 | blink_perf (two story samples), Speedometer 3.0 (the tree's `speedometer3` benchmark; it has no 3.1) | Intel | `tsan-cr-8345a0396fa5` | component build, DCHECKs on |

**Host condition of the AMD legs of 12-13 Oct (found 13 Oct).** Until 13:09 on 13 Oct, interactive sessions of
the measuring account that are not part of the benchmark were allowed on the CPUs of the host's second measuring
half (16-31, 48-63): mostly idle, at times a burst of a compile or a search for one to two minutes. Every leg on
that half before that time (Redis, FFmpeg and MySQL: core, singles and base checks) ran under this condition; it
is the same for all arms of a leg, so the ratios stand, and the legs' "other half busy" statements do not cover
it. From 13:09 those sessions are confined to the housekeeping CPUs (0-7, 32-39).

## 2. Speedup over stock TSan

Applications. Columns are the paper's configurations with the new optimizations merged in; AllOpt is the adopted
configuration. All cells but AllOpt wait for the final record legs (13-14 Oct).

| app | STC | SWMR | LO | EA | DE | DE+Peeling | DynSTC | FE-INL | AllOpt | status |
|---|---|---|---|---|---|---|---|---|---|---|
| SQLite | = stock | = stock | 1.26× [1.22, 1.31] | 0.97 [0.94, 1.00] | 0.97 [0.92, 1.00] | 0.99 [0.95, 1.03] | 1.00 [0.96, 1.03] | 1.02 [0.97, 1.05] | **1.24× [1.20, 1.28]** with DE in removal form (1.23× [1.19, 1.28] over upstream stock) | final (record cell 14 Oct 00:30-03:14, stress2, AMD, 4 offsets × N = 4, 80 of 80 runs valid, 0 void, 2 cells retired by the provenance rule (the figure over all runs is the same, 1.239), offset medians 1.256 / 1.243 / 1.199 / 1.255, A/A 0.988 [0.964, 1.009], upstream stock over the root's stock 1.006 [0.980, 1.030]; other half of the host idle in 73 of 81 cells. Statements attempted: 1.33×; contended threads 5-11 succeed 0.83× as often and attempt 1.34× as often. The verified form: record leg 12-13 Oct 1.23× [1.20, 1.27], stress2, AMD, 4 offsets × N = 4, 64 of 64 runs valid, A/A 1.000 [0.974, 1.021]; other half of the host busy in 47 of 65 cells. With the other half idle, same trees, three hours earlier: 1.28× [1.23, 1.32]; the record cell of 14 Oct, with the other half idle, reads 1.24, so the difference between 1.23 and 1.28 is not the other half: at 4 offsets × N = 4 the figure is 1.24 ± 0.04 in both conditions. LO-MARK v1; the assertion-derived spec reads the same; Intel 1.11×) DE and DE+Peeling in removal form: leg of 14 Oct, 2 offsets × N = 2, 16 of 16 valid, A/A 0.996 [0.968, 1.026]; five cells ran beside a co-load on the housekeeping CPUs, without them 0.97 [0.95, 1.00] and 1.00 [0.99, 1.02]; the verified form read 1.01 and 1.00) |
| FFmpeg | 1.00 [0.992, 1.007] | 1.00 [0.993, 1.004] | 1.00 [0.993, 1.003] | 1.00 [0.995, 1.004] | 1.01 [0.99, 1.03] | 1.01 [1.01, 1.02] | 1.28 [1.26, 1.30] | 1.00 [0.997, 1.006] | **1.27× [1.27, 1.28]** with DE in removal form, N1 placed statically and `dynstc_rt=1` only (AllOpt; 1.25× [1.24, 1.26] over upstream stock); 1.32× [1.32, 1.33] with N1 at every check (1.30× [1.29, 1.31] over upstream) | final (record cell 13-14 Oct night, four transcodes, AMD, 4 offsets × N = 4, 96 of 96 valid, A/A 1.001 [0.995, 1.007]; upstream stock over the root's stock 1.020 [1.014, 1.025]; per transcode for AllOpt: copy 2.52, mjpeg 1.03, h265 1.00, h264 1.01; other half busy in 94 of 97 cells. The verified form read the same 1.27× and 1.32× in the record cell of 13 Oct; DE and DE+Peeling in removal form with the inline test placed statically: leg of 14 Oct, 2 offsets × N = 2, 24 of 24 valid, A/A 0.996 [0.986, 1.006]; with the test at every check 1.03 [1.02, 1.03] and 1.02 [1.01, 1.03], on mjpeg 1.09 and 1.07; the verified form read 1.02 and 1.01. With `phase_guard=1` as well, the earlier record leg: 1.24× and 1.29×) |
| Redis | 1.00 [0.99, 1.01] | 0.99 [0.99, 1.00] | = stock | 1.01 [1.00, 1.02] | 1.11 [1.08, 1.13] | 1.08 [1.06, 1.10] | 1.14 [1.10, 1.19] | 1.07 [1.06, 1.07] | **1.47× [1.45, 1.49]** with DE in removal form (1.43× [1.40, 1.45] over upstream stock) | final (record cell 13 Oct evening, seven commands, AMD, 4 offsets × N = 4, 104 of 104 valid, A/A 1.003 [0.993, 1.010]; the verified form in the same cell 1.48 [1.42, 1.52], removal over verified 1.00 [0.97, 1.03]; DE and DE+Peeling in removal form with the inline test, 2 offsets × N = 4; other half busy in 91 of 105 cells and tree copies to the host's disk beside the leg. The morning's record leg read 1.52× [1.49, 1.53] for the verified form) |
| MySQL | 1.00 [0.993, 1.008] | 1.00 [0.994, 1.003] | 1.00 [0.996, 1.006] | 1.01 [1.00, 1.01] | 1.00 [0.99, 1.00] | 0.99 [0.99, 1.00] | 1.00 [0.992, 1.002] | 1.09 [1.08, 1.09] | 1.10× [1.09, 1.11] | final (singles leg 13 Oct: 2 offsets × N = 2, 40 of 40 valid, A/A 0.995 [0.992, 0.999]; FE-INL per script: delete 1.13, insert 1.08, update 1.06. Record leg 13 Oct, three write scripts, `-O2 -g`, AMD, 4 offsets × N = 4, 64 of 64 valid, A/A 1.001 [0.997, 1.004]; other half of the host busy in 97 of 101 cells. Per script: delete 1.14, insert 1.09, update 1.07) |
| memcached | 1.00 [0.999, 1.006] | 1.01 [1.012, 1.016] | 1.00 [0.995, 1.003] | 1.86 [1.84, 1.87] | 1.00 [1.00, 1.01] | 1.00 [0.99, 1.02] | 1.00 [0.995, 1.003] | 1.00 [1.000, 1.004] | **1.91× [1.90, 1.92]** with DE in removal form (1.82× [1.81, 1.83] over upstream stock) | final (record cell 13-14 Oct night, AMD, 4 offsets × N = 4, 80 of 80 valid, 0 void, A/A 1.001 [0.998, 1.006]; offset medians 1.900 / 1.909 / 1.909 / 1.911; other half of the host busy in 79 of 81 cells. The verified form read 1.89× [1.89, 1.90] in the record leg of 13 Oct, 1.81× over upstream stock; DE and DE+Peeling in removal form: leg of 14 Oct, 2 offsets × N = 2, 16 of 16 valid, 0 void, A/A 0.993 [0.984, 1.001], read under a declared co-load on the housekeeping CPUs (threshold 0.40; at 0.10 fourteen of sixteen cells would be retired); the verified form read 0.98 and 1.00. See the note on the base below) |

**The four analyses together** (STC + SWMR + LO + EA, each as in its single-configuration column, no DE, no
peeling, no DynSTC, no FE-INL), over stock, 2 offsets × N = 2:

| app | speedup | status |
|---|---|---|
| FFmpeg | 1.00 [1.00, 1.01] (copy 1.01, h264 1.01, h265 1.00, mjpeg 1.00; A/A 1.00 [0.99, 1.00]; 12 of 12 valid) | final (14 Oct) |
| SQLite | 1.23 [1.18, 1.27] (offsets 1.250 / 1.207; A/A 0.99 [0.95, 1.04]; 12 of 12 valid, 0 void; housekeeping CPUs quiet) | final (14 Oct): the four analyses carry the whole figure of the adopted line (1.24) |
| Redis | 0.99 [0.98, 1.00] (offsets 0.983 / 0.992; A/A 1.00 [0.98, 1.01]; 12 of 12 valid; MSET 0.98, ZPOPMIN 0.95 [0.90, 1.00], the other five commands 0.99-1.00) | final (14 Oct) |
| memcached | | pending (14 Oct, about 11:00) |
| MySQL | not built | |

**The base.** "Stock" in this table is the stock program built by the same compiler root with every optimization
off and run on that root's runtime. For memcached that base is 4.7 % slower than upstream LLVM's TSan (leg of
13 Oct: upstream stock over the root's stock 1.049 [1.046, 1.051], A/A 1.001), so AllOpt is 1.89× over the root's
stock and 1.81× [1.80, 1.81] over upstream stock, and upstream stock is about 8.05× slower than native. The known
cause is the root's runtime (the ownership-marker test in the access slow path is always on; 8 Oct: 0.895 of
upstream before a fix, 0.959 after); a leg that separates compiler from runtime is queued. FFmpeg (leg of 13 Oct, 2 offsets × N = 2, 16 of 16 valid): the root's stock over upstream stock 0.981
[0.973, 0.986], so 1.9 % slower; the line with N1 at every check reads 1.26× [1.24, 1.27] over upstream stock
(1.29× over the root's); the adopted line over upstream stock: pending (in the corrected-arm leg). SQLite (leg of 13 Oct, 2 offsets × N = 2, 16 of 16 valid, three cells disturbed): not resolved at this size, the
root's stock over upstream stock 0.974 [0.940, 1.009] with a second copy of the root's stock at 0.954; AllOpt over
upstream stock 1.25× [1.22, 1.29] (undisturbed pass alone 1.23×), so the base does not move SQLite's figure beyond
its interval; upstream stock is an arm of the idle-half recheck. Redis (leg of 13 Oct, 2 offsets × N = 2, 16 of 16 valid): the root's stock over upstream stock 0.979 [0.969,
0.993] (a second copy 0.982), so 2 % slower; AllOpt over upstream stock 1.47× [1.42, 1.49] (1.52× over the root's).
MySQL (leg of 13 Oct evening, 2 offsets × N = 2, 20 of 20 valid): upstream stock over the root's stock 1.001
[0.997, 1.006], no base difference. In the same leg, DE in removal form against the verified form: 0.994 [0.990,
0.998], so 0.6 % slower with an interval that excludes 1; over stock the verified form 1.105 [1.100, 1.110], removal
1.099 [1.092, 1.104], A/A 0.999 [0.995, 1.004]. Which base the paper states: to be decided; until then every figure
above is over the root's own stock.

**Cells under correction (audit of 13 Oct, `audits/audit-a89-mapping-vs-arms.md`).** The single-configuration trees
of 12-13 Oct were built from one template, not from each application's adopted line, so some columns do not contain
the optimization the mapping reports under them. Being replaced by new trees (built 13 Oct, preservation running,
not timed yet): Redis DynSTC: REPLACED 13 Oct, quiet threads with the phase guard and without N1 reads 1.14 [1.10, 1.19] (separate leg, 2 offsets × N = 2, 16 of 16 valid, A/A 0.985 [0.962, 1.007], AllOpt 1.49 in that leg; MSET 1.38, the other commands 1.04-1.17; the run-time option `dynstc_rt` alone read 1.00); memcached EA: REPLACED 13 Oct, EA with EVCONF-CHECKED reads 1.86 [1.84, 1.87] (separate leg, 2 offsets × N = 2,
20 of 20 valid, 0 void, A/A 1.004 [0.999, 1.007]; AllOpt 1.90 in that leg, the arm over AllOpt 0.978 [0.971, 0.984];
EA without it 1.00; server CPU 963 s against stock's 1,888 s; preservation 10 against 10: stock's 4 sites in 10 of 10 runs in this arm, 0 void); FFmpeg DynSTC, DE and DE+Peeling: REPLACED 13 Oct (leg of 2 offsets × N = 2, 48 of 48 valid, A/A 1.002 [0.991,
1.016]): DynSTC = `dynstc_rt=1` with N1-ST and statically placed N1, 1.28 [1.26, 1.30] (the run-time option alone
read 1.14); DE and DE+Peeling with statically placed N1, 1.02 and 1.01 (with N1 at every check 1.02 and 1.03).
**FFmpeg's options, same leg:** `phase_guard=1`, which the adopted line carried although FFmpeg has no phase code,
costs the h264 transcode 5-6 % (0.947 with it, 0.998-1.015 without). The adopted line with `dynstc_rt=1` only
reads 1.27× [1.26, 1.29] over the root's stock and 1.26× [1.24, 1.28] over upstream stock, against 1.25× and 1.23×
with both options; N1 at every check with `dynstc_rt=1` only 1.32× [1.30, 1.34]. The 4 × 4 record cell with
`dynstc_rt=1` only is queued; until it is in, the table keeps 1.24×. The root's stock over upstream stock in this
leg: 0.990 [0.972, 1.009]. Stated as they are, not replaced: the DynSTC column of
SQLite, memcached and MySQL is the run-time form (`dynstc_rt=1`), which their adopted lines do not use; the EA and
SWMR columns of SQLite, FFmpeg and MySQL carry MEMINTR and SWMR-ROOTS, which their adopted lines do not use; FE-INL
is part of AllOpt only for Redis, MySQL and Chromium; SQLite's adopted line is built with DE and peeling. The
MySQL figure of 1.13× quoted earlier was over upstream stock, the 1.10× here is over the root's own stock.

Single configurations: 2 offsets × N = 2 (core arms 4 × 4), so their intervals are wide; the A/A of SQLite's
singles leg reads 1.014 [0.938, 1.073], and every SQLite column but LO is inside it. Redis's singles leg: A/A 1.005
[0.998, 1.012]; DE, DE+Peeling and FE-INL are above it, the rest within 1 %; its gain is in quiet threads, which
only AllOpt contains (preservation of Redis's single configurations: pending). FFmpeg's singles leg: A/A 0.998 [0.991, 1.004]; STC, SWMR,
LO, EA and FE-INL are inside 0.992-1.007; DynSTC is the run-time option on the stock binary (copy 1.79, h264 0.93);
DE's gain is on mjpeg (1.10). The compiled code adds 1.09 on top of the option in the adopted line (1.238 / 1.138) and 1.13 with N1 at every check, on the copy transcode. memcached's singles leg: A/A
1.002 [0.998, 1.005]; only SWMR is resolved above it (1.4 %); DE's 0.98 comes from one of the two offsets; the gain
is EVCONF-CHECKED (the run-time form of EA), which only AllOpt contains. "= stock": the build is
byte-identical to stock and was not timed.

FFmpeg, per transcode in the final leg (N1 at every check / N1 placed statically, over stock; stock's slowdown
over native): copy 2.71 / 2.44; 5.1× (native takes 0.11 s on a 0.01 s clock, coarse); mjpeg 1.08 / 1.04; 6.3×;
h265 1.00 / 1.00; 1.42×; h264 0.94 / 0.93; 1.29×. The static line is 0.963 [0.955, 0.971] of the other and costs
+18 % compile time against about +100 %. In the leg of 12 Oct the run-time option alone on the stock binary read
1.135×, and h264 was 7-8 % slower in every arm that had the option.

Chromium, Intel, N = 3 repetitions, speedup over stock:

| configuration | first sample (49 stories) | hold-out sample (48 stories) | status |
|---|---|---|---|
| the four analyses (STC, SWMR, LO, EA) | 1.005 | — | final |
| + DE | 1.012 | — | final |
| + peeling | 0.993 | — | final |
| the four analyses + FE-INL + MEMINTR | 1.104 | 1.101 | final |
| AllOpt (+ N1) | 1.286 | 1.248 | final |

Noise floor: a layout twin of the FE-INL configuration reads 1.013 over it. AllOpt on the AMD host: 1.081 and 1.017.

Per story, Intel, tabulated 14 Oct from the same runs (intervals: bootstrap over stories, 10,000 draws; per-story
tables for the plot under `chromium/legs/story-tables/` of the measuring tree):

| | hold-out, FE-INL configuration | hold-out, AllOpt | first sample, FE-INL configuration | first sample, AllOpt (one repetition on the corrected harness) |
|---|---|---|---|---|
| geometric mean | 1.101 [1.086, 1.116] | 1.248 [1.199, 1.299] | 1.103 [1.084, 1.124] | 1.292 [1.236, 1.354] |
| median | 1.107 [1.099, 1.113] | 1.260 [1.177, 1.288] | 1.093 [1.085, 1.119] | 1.269 [1.221, 1.328] |
| quartiles | 1.081-1.123 | 1.147-1.317 | | |
| range | 0.991-1.223 | 0.983-1.867 | 0.992-1.375 | 0.981-2.012 |
| stories below 1.00 / at least 1.10 / 1.25 / 1.50 | 2 / 30 / 0 / 0 | 1 / 39 / 25 / 6 | | |
| per suite, median: dom (3 stories) | 1.030 | 1.259 | 1.085 | 1.114 |
| image_decoder (2) | 1.010 | 1.038 | 1.014 | 1.045 |
| layout (29) | 1.112 | 1.288 | 1.110 | 1.321 |
| paint (2) | 1.079 | 1.211 | 1.051 | 1.601 |
| parser (7) | 1.033 | 1.044 | 1.070 | 1.268 |
| svg (5) | 1.120 | 1.268 | 1.079 | 1.253 |

Status: final for the hold-out sample (N = 3) and for the FE-INL configuration on the first sample; current for
AllOpt on the first sample (N = 1). Story counts per suite are those of the hold-out sample; four suites have 2 to
7 stories, so their medians are indicative only.

The same tabulation on the AMD host (existing runs, N = 3; the paper takes the Intel figures, decided 11 Oct):

| | first sample, FE-INL configuration | first sample, AllOpt | hold-out, FE-INL configuration | hold-out, AllOpt |
|---|---|---|---|---|
| geometric mean | 1.063 [1.053, 1.075] | 1.077 [1.036, 1.123] | 1.057 [1.045, 1.070] | 1.017 [0.992, 1.047] |
| median | 1.058 [1.048, 1.070] | 1.027 [1.002, 1.053] | 1.054 | 0.988 [0.977, 1.011] |
| range | 1.004-1.192 | 0.920-1.746 | 0.918-1.186 | |
| stories below 1.00 | 0 of 49 | 17 of 49 | 3 of 48 | 27 of 48 |
| per suite, median: dom / image_decoder / layout / paint / parser / svg | 1.055 / 1.023 / 1.069 / 1.067 / 1.027 / 1.041 | 0.985 / 1.056 / 1.027 / 1.655 / 1.006 / 1.019 | 1.062 / 1.026 / 1.066 / 1.053 / 1.014 / 1.046 | 1.017 / 1.023 / 0.987 / 1.047 / 1.004 / 0.972 |

The FE-INL configuration is a narrow distribution on both hosts (about 1.10 on Intel, 1.06 on AMD). AllOpt depends
on the host: on Intel the median story gains 1.26 and one story is below stock; on AMD the median story is at 1.03
and 0.99, and a third to more than half of the stories are slower than stock. final (as a tabulation).

Speedometer 3, Score over stock, N = 3 repetitions (range):

| configuration | Intel | AMD | status |
|---|---|---|---|
| the four analyses + FE-INL + MEMINTR | 1.112 (1.082–1.158) | 1.082 (1.049–1.147) | final |
| AllOpt (+ N1) | 1.225 (1.201, 1.250; N = 2, one run failed) | 1.006 (0.983–1.048) | final |
| A/A | 1.012 (0.994–1.044) | 1.009 (0.972–1.042) | final |

Second Intel leg with a native arm (14 Oct 02:43-06:53, N = 3, 12 of 12 runs valid, host quiet, no A/A), Score over
stock per repetition and geometric mean: the four analyses + FE-INL + MEMINTR 1.066 / 1.045 / 1.073 = 1.06; AllOpt
1.200 / 1.113 / 1.133 = 1.15; native 9.14 / 9.78 / 9.25 = 9.4. Slowdown over native on Speedometer 3.0: stock 9.4×,
FE-INL configuration 8.8×, AllOpt 8.2×. Over the 20 per-test rows (bootstrap over tests): 1.05 [1.02, 1.09], 1.13
[1.09, 1.17], native 7.8 [6.9, 8.9]. The gains are lower than in the first leg (1.11, 1.23); the two legs are not
pooled, so on Intel Speedometer reads 1.06-1.11 and 1.15-1.23. Memory, mean of 3: sampled peak of the browser's
cgroup (anonymous + shared) native 0.96 GiB, stock 7.46, FE-INL configuration 7.58, AllOpt 7.91: stock 7.8× native,
memory overhead reduction −1.8 % and −6.9 % (an increase); largest process's peak resident memory native 0.36 GiB,
stock 1.20, AllOpt 1.88. CPU time of the browser was not recorded in this leg. final.

On Intel one repetition of the FE-INL configuration ran on a disturbed host; without it the figure is 1.090 (N = 2).
Three of about 46 Speedometer runs of AllOpt died on a debug check of V8's garbage collector; no other build did; the
compiled code points to a timing-dependent condition in V8. Confirmed: with V8's forced compaction the stock TSan
build dies on the same check (1 of 5 runs; the FE-INL configuration 1 of 4). Whether AllOpt hits it more often than stock in normal runs
is not established: 3 of about 46 (about 6.5 %, rough 95 % range 1.4-18 %) against 0 of about 30 (upper bound
about 10 %); in the counting loops of 12-13 Oct, the two builds in turn, AllOpt 1 of 19 and stock 0 of 19.

## 3. Slowdown over native, memory

Slowdown = figure of merit of native over the build's. Memory: peak resident memory of the timed process; the
reduction is (R_stock − R_opt) / R_stock with R = peak − native's peak. Other applications: pending (final legs;
Chromium needs a native build).

| app | stock / native | AllOpt / native | peak RSS native / stock / AllOpt | memory overhead reduction | status |
|---|---|---|---|---|---|
| SQLite | 5.22× [5.06, 5.48] | 4.22× with DE in removal form (verified form 4.40× against 5.43× in the leg of 12-13 Oct) | 8 / 60 / 61 MiB | about −2 % (an increase) | final (record cell 14 Oct); CPU per run native 168 s, stock 145 s, AllOpt 148 s |
| Redis (server process) | 6.21× | 4.22× with DE in removal form (verified form 4.07× in the morning's leg, 4.20× in this cell) | 46 / 965 / 970 MiB | −0.5 % (an increase) | final (record cell 13 Oct evening); server CPU per run 6.4 / 175.0 / 108.8 s |
| memcached (server process) | 8.48× [8.39, 8.58] | 4.44× with DE in removal form (verified form 4.46× against 8.44× in the leg of 13 Oct) | 27 / 108 / 100 MiB | 10 % | final (record cell 13-14 Oct night); server CPU per run 221 / 1833 / 904 s; upstream stock 8.09× slower than native, 1750 s, 108 MiB |
| MySQL (server process, sampled every 5 s) | 10.03× [9.94, 10.15] | 9.11× | 583 / 3394 / 3413 MiB | −0.7 % (an increase) | final; server CPU per cell 1341 / 2181 / 2132 s; per script stock is 17.7× (delete), 8.7× (insert), 6.6× (update) slower than native |
| FFmpeg, N1 at every check | 2.76× [2.75, 2.79] (upstream stock 2.71×) | 2.09× with DE in removal form | 521 / 1440 / 1465 MiB (this cell, 16 runs per arm) | −2.7 % (an increase) | final (record cell 13-14 Oct night, `dynstc_rt=1` only) |
| FFmpeg, N1 placed statically | 2.76× [2.75, 2.79] | 2.17× with DE in removal form | 521 / 1440 / 1441 MiB (this cell, 16 runs per arm) | −0.1 % (no reduction) | final (record cell 13-14 Oct night, `dynstc_rt=1` only); CPU per run native 373 s, stock 711, AllOpt 707 |

Chromium, Intel hold-out, AllOpt (CR1+N1) over stock per suite, min / q1 / median / q3 / max (recount of 14 Oct):
dom (3 stories) 1.018 / 1.139 / 1.259 / 1.410 / 1.561; image_decoder (2) 1.026 / 1.032 / 1.038 / 1.045 / 1.051;
layout (29) 0.983 / 1.224 / 1.288 / 1.373 / 1.867; paint (2) 1.161 / 1.186 / 1.211 / 1.235 / 1.260; parser (7)
1.000 / 1.028 / 1.044 / 1.100 / 1.185; svg (5) 1.130 / 1.185 / 1.268 / 1.302 / 1.413; all 48: 0.983 / 1.147 / 1.260 /
1.317 / 1.867. CDF points: `/extra/alexey/chromium/legs/story-tables/hold-intel-cdf-cr1n1.tsv`. final.

Chromium, native on the Intel hold-out sample (14 Oct, 6 of 6 runs valid; stock from the hold-out runs of 11-12 Oct,
same binary, pinning and harness, another session): stock is 13.4× [11.4, 15.4] slower than native as a geometric
mean over the 48 stories (median 15.5, range 1.9-25.0); per suite, geometric mean (median): dom 11.6 (15.4),
image_decoder 6.0 (7.0), layout 14.1 (16.0), paint 5.4 (5.9), parser 18.8 (20.9), svg 13.3 (13.8). With the
hold-out gains: FE-INL configuration 12.1×, AllOpt 10.7× slower than native. Memory, stock over native: sampled
peak of the browser's cgroup 4.3× (3.2-6.6 over the six suites); largest process 3.6×. No CPU ratio (the stock runs
predate the record). final.

Chromium, largest process's peak resident memory over stock: FE-INL configuration 1.06×, AllOpt 1.54× on Intel
(Speedometer); 1.09× and 1.77× on AMD (mapped code; allocated memory unchanged). current.

## 4. Instrumentation reduction and code size

Counted in the final legs' binaries (offset-00 trees, 13 Oct; call sites of the runtime's entries from outside the
runtime). Q_call = accesses checked by a plain runtime call; inline = accesses checked by an inline test; guarded =
accesses behind a run-time guard; Q = their sum. "Runtime calls avoided" = (Q_call,stock − Q_call) / Q_call,stock.
Text size includes the statically linked runtime.

| app | AllOpt | Q_call | inline | guarded | Q (stock's Q) | Q over stock | runtime calls avoided | text over stock | status |
|---|---|---|---|---|---|---|---|---|---|
| SQLite | LO-MARK v1, DE in removal form | 70,516 | 0 | 1,836 | 72,352 (57,997) | +24.8 % | −21.6 % | 1.14× | final |
| memcached | EVCONF-CHECKED, DE in removal form | 6,525 | 0 | ≥ 313 | ≥ 6,838 (6,748) | ≥ +1.3 % | +3.3 % | 1.04× | final (owner checks: lower bound) |
| Redis | adopted, DE in removal form | 3,484 | 37,706 | 19 | 41,209 (37,941) | +8.6 % | +90.8 % | 2.55× | final |
| FFmpeg | N1 placed statically, DE in removal form | 509,974 | 39,265 | 0 | 549,239 (514,609) | +6.7 % | +0.9 % | 1.26× | final |
| FFmpeg | N1 at every check, DE in removal form | 65,086 | 484,774 | 0 | 549,860 (514,609) | +6.9 % | +87.4 % | 3.06× | final |
| MySQL | FE-INL line (DE in its verified form, MySQL's adopted form) | 638,391 | 26,566 | 0 | 664,957 (599,000) | +11.0 % | −6.6 % | 1.31× | final |
| Chromium, stock | | 13,588,706 | 0 | 0 | 13,588,706 | | | 773.7 MB | final |
| Chromium | the four analyses + FE-INL + MEMINTR | 13,483,741 | 0 | 0 | 13,483,741 (13,588,706) | −0.8 % | +0.8 % | 1.18× | final |
| Chromium | AllOpt (+ N1) | 1,173,107 | 12,331,507 | 0 | 13,504,614 (13,588,706) | −0.6 % | +91.4 % | 2.99× | final |

Recount of 14 Oct from the binaries, after DE took its removal form. A covering write kept by removal DE is a
plain runtime call and counts in Q_call. Against the verified form (Q over stock; text over stock): SQLite +27.0 →
+24.8 %, 1.18 → 1.14×; memcached ≥ +5.3 → ≥ +1.3 %, 1.06 → 1.04×; Redis +10.5 → +8.6 %, 2.58 → 2.55×; FFmpeg
static +13.4 → +6.7 %, 1.40 → 1.26×; FFmpeg every check +13.5 → +6.9 %, 3.24 → 3.06×. The adopted lines still
carry more checks than stock in every application: peeling and SQLite's function copies duplicate checks.
Chromium: counted over the main binary and all 486 libraries of the existing builds, loaded or not; an inline
test is counted by its slow-path call site.

Single configurations with DE in removal form, Q over stock (verified form in brackets) and text over stock:

| app | DE | DE+Peeling |
|---|---|---|
| SQLite | −1.7 % (+0.1), 1.00× | +8.6 % (+10.8), 1.05× |
| memcached | −3.6 % (0.0), 1.00× | +7.8 % (+12.1), 1.02× |
| Redis (with the inline test) | −1.4 % (+0.1), 2.04× | +15.3 % (+17.4), 2.27× |
| FFmpeg, N1 placed statically | −4.5 % (0.0), 1.20× | +9.2 % (+16.0), 1.26× |
| FFmpeg, N1 at every check | −4.4 % (+0.1), 2.71× | +9.3 % (+16.2), 3.05× |

DE alone is the one configuration whose check count falls below stock, by 1.4 to 4.5 %. final.

Checks by form from the pass's statistics with DE in removal form exist for SQLite's `sqlite3.c` only (56,636
checks in stock; EA removes 625, as in the verified form; DE removes 1,161: 865 by dominance, 296 by
post-dominance; kept in any form 69,466 against 70,627 verified; 1,836 lock-ownership marks). For FFmpeg, Redis and
memcached the table of removals by analysis below is still that of the verified form.

**These are counts of checks in the binary, not the paper's static reduction, and they are not below stock.** Two
causes, both visible in the single-configuration rows (`experiments.md`, "M4 and M7 on the final legs' binaries"):
verified DE keeps a check as an inline test where the submitted DE removed it, so it moves accesses out of Q_call
without lowering Q; loop peeling and SQLite's function copies duplicate code together with its checks (DE+Peeling
adds 11-17 % to Q). The single analyses that remove checks are small here: EA 0.6-5.9 %, STC up to 3.2 %, SWMR up
to 2.7 %. The pass's own statistics follow.

**Checks by form, from the pass's statistics** (13 Oct; each arm's flag line compiled once with statistics saved
per object, summed over the program; counts are after duplication by peeling and function copies, which the pass
cannot tell apart, so they are not per source access):

| app | checks in stock | removed by the paper's analyses (STC / SWMR / LO / EA) | share of stock | AllOpt: plain call | AllOpt: inline test | AllOpt: behind a run-time guard | status |
|---|---|---|---|---|---|---|---|
| SQLite | 58,778 | 835 (0 / 0 / 0 / 835) | 1.4 % | 70,417 | 910 | 1,871 (lock-ownership marks) | final |
| FFmpeg, N1 placed statically | 522,110 | 14,085 (0 / 0 / 0 / 14,085) | 2.7 % | 522,609 | 70,644 | 0 | final |
| Redis | 38,557 | 2,752 (920 / 60 / 0 / 1,772) | 7.1 % | 3,232 | 39,361 | 0 | final |
| MySQL | 993,181 | 6,250 (0 / 0 / 0 / 6,250) | 0.6 % | 1,037,536 | 42,417 | 0 | final |
| memcached | 6,911 | 860 (221 / 239 / 13 / 387) | 12.4 % | 6,846 | 213 | 527 (owner checks) | final |

SQLite, FFmpeg and MySQL are compiled unit by unit, where STC, SWMR and LO remove nothing; Redis and memcached use
the whole-program step. Peeling adds copies of checks: 677 loops peeled in SQLite, 8,834 in FFmpeg, 21,029 in
MySQL, about 7,190 copied checks in Redis and 974 in memcached. Control: every single configuration that does not
duplicate code sees exactly stock's number of accesses.

**Checks entering the runtime, per operation** (13 Oct; processor trace of all threads of the process for 0.25 s
under the leg's load; calls from program code into the runtime's access entries per call of a once-per-operation
function; a check that makes no call — an inline hit, a skipped check, an inline owner or lock test — is not
counted, so this is the drop in runtime entries, not the paper's dynamic reduction; one window per test, counts,
not times; method checked on 7 Oct against a run-time census to 0.1 %):

| app | stock | AllOpt | drop | status |
|---|---|---|---|---|
| memcached, per request | 1,887.4 | 429.0 | 77.3 % | final (14 Oct, DE in removal form; verified form 435.3, 76.9 %) |
| Redis, weighted over the seven commands | | | 93.6 % | final (14 Oct, DE in removal form; verified form 94.5 %) |
| Redis LRANGE_100 / _300 / _500 / _600, per command | 508 / 23,904 / 39,538 / 47,361 | 51 / 1,518 / 2,520 / 3,026 | 90.0 / 93.7 / 93.6 / 93.6 % | final (removal form; verified form 50 / 311 / 510 / 612) |
| Redis ZADD / ZPOPMIN / MSET, per command | 510 / 339 / 2,442 | 24 / 37 / 82 | 95.3 / 89.0 / 96.7 % | final (removal form; verified form 47 / 32 / 48) |

Redis in removal form enters the runtime more often than the verified form on LRANGE and MSET: the covering write
of a removed pair is made by a runtime call (about 1,200 per LRANGE_300 command) that gets neither the inline test
nor the phase skip. ZADD goes the other way. The timed figures of the two forms are equal (1.47× and 1.48×).
MySQL keeps the verified form; its rows of 13 Oct stand. Only runtime entries are counted: the paper's L, checks
executed in any form, would need counting code in the pass or the runtime and is not measured.

FFmpeg and SQLite have no once-per-operation call, so their unit is runtime entries per 1,000 function entries of
instrumented code (three windows per transcode, one per SQLite test):

| app | test | drop in runtime entries, AllOpt against stock | status |
|---|---|---|---|
| FFmpeg (static line, DE in removal form, `dynstc_rt=1` only) | stream copy / mjpeg / h264 / h265 | 85.0 % / 18.0 % / 5.1 % / 1.5 % (pooled over each transcode's windows; means of windows 85 / 18.6 / 5.0 / 2.2 %) | final (14 Oct; verified form with both options about 78 % / 23 % / 5 % / 3 %) |
| MySQL (FE-INL line, verified form; record write set, 24 connections, runtime entries per statement) | insert / update_non_index / delete | +2.5 % (a rise) / −9.2 % / +2.6 % (a rise); composite weighted by stock entries per second +1.7 % (a rise): about 0 % | current (14 Oct; one 0.25 s window per script, 255-356 statements in the insert and update windows, so a few percent is not resolved. Function entry and exit calls per statement fall from 49,525 / 52,360 / 16,281 to 112 / 104 / 38: that is what the line removes on MySQL) |
| FFmpeg, composite over the four transcodes | weights: stock runtime entries over the whole run (shares: copy 1.1 %, mjpeg 50.7, h264 32.6, h265 15.5) | 12.0 % | current (recount of 14 Oct: a transcode's rate from windows at 1, 4 and 8 s is assumed to hold for its whole run; the encoder libraries are not instrumented) |
| SQLite | stress2 (the record workload) | 9.9 % (15,486 → 13,952) | final (14 Oct, DE in removal form; verified form 11.8 %) |
| SQLite | the other thirteen tests of threadtest3 | −5.6 % to 6.2 %; within 4 % of stock in ten of them | final (14 Oct, removal form) |

FFmpeg agrees with its timing: the gain is in the stream copy. **SQLite does not:** a drop of 12 % in runtime
entries on stress2 does not by itself account for 1.23-1.28×. Checked 13 Oct: the figure itself reproduces outside
the harness (same command, fresh directory, three alternating runs: 38.0k against 47.7k statements, 1.26×). A
sampling profiler attached to the process removes the difference (1.03×: stock 9 % faster, the adopted line 11 %
slower), so a sampled profile of this lock-heavy test is not a profile of the measured run and is not used; a
counting run was taken instead, and **it removes the gain as well**: under `perf stat` (counters only, no
sampling) the record command reads 1.00 (stock 42.5-44.4k, the adopted line 42.4-43.0k) and stress2 alone 1.08;
in both perf modes stock gets about 12 % faster and the adopted line 9-11 % slower. Without perf the gain is there
in every form tried on 13 Oct: 1.26 (bare command), 1.22 (through the test script), 1.23 and 1.28 (record legs).
The test makes about 53,000 voluntary context switches a second and keeps about 1.2 of its 16 CPUs busy: it is
governed by hand-offs of the database lock, and work added to each context switch is enough to erase the
difference. From the final leg's records total CPU is equal in both arms (145.0 and 147.7 s) and statements are
37,544 and 46,522. **How much of the gain is cheaper checks under the lock and how much is a change in who wins
the lock is not measured.** The figure is reproducible but sensitive to scheduling; a robustness series over
pinnings without perf is running.

**What the SQLite figure counts** (13 Oct, from the per-thread "ok/attempted" lines of the two undisturbed
sessions, 3 runs per arm each): "statements OK" rises 38,001 → 47,731 (1.26×), carried by four threads that run
trivial statements (PRAGMA integrity_check, CREATE TABLE IF NOT EXISTS, DROP TABLE IF EXISTS, a second PRAGMA
loop: 1.30-1.37× each); the two SELECT threads gain 1.12-1.14×; **the seven data-modifying threads (INSERT,
UPDATE, DELETE, VACUUM) succeed less often: 3,493 → 2,925 statements OK in sum (0.75-0.88× each).** What rises
uniformly is the attempt rate: every contending loop attempts 1.33-1.37× as many statements (141,978 → 190,127
in sum, 1.34×), about three quarters of which fail on a lock. With about one CPU busy that is CPU per attempt
0.44 → 0.33 ms (25 % less). So the figure as quoted is a count dominated by trivial statements; the paper must
name the metric (statements OK, or attempts) and state that successful data-modifying statements fall.

**Robustness of SQLite's figure without perf** (13 Oct, 3 alternating runs per arm per variant, bare command):

| variant | AllOpt over stock, statements OK (three pairs) | stock: OK of attempts | AllOpt: OK of attempts |
|---|---|---|---|
| record pinning, 16 logical CPUs | 1.26 (1.27 / 1.20 / 1.30) | 38.0k of 142k | 47.7k of 190k |
| 8 CPUs, one per core | 1.25 (1.22 / 1.26 / 1.27) | 40.1k of 152k | 50.1k of 201k |
| 4 CPUs | 1.27 (1.24 / 1.29 / 1.27) | 43.4k of 165k | 55.0k of 218k |
| 2 CPUs | 1.06 (1.04 / 1.04 / 1.09) | 28.8k of 50.9k | 30.4k of 54.3k |
| record pinning, stress2 alone | 1.22 (1.22 / 1.25 / 1.18) | 38.9k of 147k | 47.4k of 190k |

CPU is the same in both arms in every variant. The figure holds (1.22-1.27) wherever the 15 threads have CPUs to
contend for the lock, and is small (1.06) at 2 CPUs, where they queue for a CPU instead and attempts fall to a
third. A probe that only reads /proc (no perf) on an undisturbed run of stress2 alone: 1.09 and 1.10 CPUs busy,
CPU per attempted statement 0.444 ms against 0.351 ms (21 % less), voluntary context switches per attempt 13.5
against 11.0. So the adopted line spends 21 % less CPU per attempt, attempts 28 % more statements with the same
total CPU, and completes 22 % more. Two perturbations remove the gain: a perf attachment, and fewer than about
four CPUs. current.

**In a timed leg, with DE in removal form beside the verified form** (13 Oct, 2 offsets × N = 2, 16 of 16 valid, 0
void, A/A 1.006 [0.984, 1.038]): over stock, removal 1.27 [1.25, 1.29] and verified 1.27 [1.24, 1.29]; removal
over verified 1.00 [0.99, 1.03]. Statements OK of attempted, mean per run: stock 37,592 of 137,831; verified 47,638
of 186,701; removal 47,841 of 187,640. By thread class, stock → removal: the always-successful loop 12,362 →
16,498; the three half-successful loops about 5.75k of 11.2k → 7.9k of 15.5k each; the seven contended loops
438-628 of 11.4-14.6k → 350-530 of 15.8-19.7k. Every loop attempts about 35 % more; the loops that mostly succeed
complete 20-40 % more; the contended loops complete 10-20 % fewer. current.

MySQL, runtime entries per statement, stock → AllOpt (one window per workload): oltp_read_write 53,454 → 46,011
(13.9 % fewer); oltp_read_only 58,146 → 48,466 (16.6 %); oltp_write_only 27,660 → 27,456 (0.7 %);
select_random_ranges 2.8 %; select_random_points 4.2 %. Calls at function entry and exit per statement fall from
42-109 thousand to 91-228: they are inlined, which is where MySQL's gain comes from. current.
Chromium code size over stock: FE-INL configuration 1.18×, AllOpt 3.0× (2.99×), over the main binary and all libraries. final.

## 5. Compile time

Over the stock TSan build; fresh trees, medians of 3, wall seconds.

**Against the native build** (the submitted paper's definition; 13 Oct, one quiet window, 3 builds per arm; the
adopted medians of Redis and SQLite are from earlier windows, so those two ratios are across windows):

| app | native | stock TSan | adopted | stock over native | adopted over native | status |
|---|---|---|---|---|---|---|
| memcached | 6.85 s | 9.32 s | 16.32 s | +36 % | +138 % | final |
| FFmpeg (N1 placed statically) | 65.23 s | 76.44 s | 89.70 s | +17 % | +38 % (N1 at every check: 136 s, +108 %) | final |
| Redis | 4.96 s | 5.29 s | 14.31 s | +7 % | +189 % | current (across windows) |
| SQLite | 22.20 s | 25.25 s | 37.53 s | +14 % | +69 % | current (across windows) |
| MySQL | 863.4 s | 925.0 s | 1,075.9 s | +7 % | +25 % (+16 % over stock) | current (means of 2 builds per arm, 13 Oct; Release, the verified form of DE) |

**Final window, DE in removal form** (14 Oct 08:00-08:29, host quiet apart from the builds, adopted removal lines;
native, stock and adopted alternating, 3 builds per arm, medians, 8 jobs; spread per arm within 3.3 %). These rows
replace the two tables around them:

| app | native | stock TSan | adopted | stock over native | adopted over native | adopted over stock | status |
|---|---|---|---|---|---|---|---|
| Redis | 4.82 s | 5.09 s | 15.08 s (whole-program step 4.61 s + build 10.48 s) | +6 % | +213 % | +197 % | final |
| SQLite | 22.31 s | 24.93 s | 38.64 s | +12 % | +73 % | +55 % | final |
| FFmpeg (N1 placed statically) | 64.10 s | 74.62 s | 84.86 s | +16 % | +32 % | +14 % | final |
| memcached (configure run once) | 6.84 s | 9.40 s | 11.38 s (whole-program step with the one configure 9.12 s + build 2.23 s) | +37 % | +66 % | +21 % (19-21 % over four sets of nine builds; with configure run twice it read +76 %) | final (quote the ratio: configure ran in a slower mode in this window in every arm, so the seconds are this window's) |
| MySQL (Release, verified form, 6 jobs) | 853.8 s | 920.3 s | 1067.8 s | +8 % | +25 % | +16 % | final (rounds 1-2 of 13 Oct and round 3 of 14 Oct; per arm 820-873, 858-930, 1018-1084 s) |
| Chromium | see the row below | | | −1 % | +10 % | +11 % | current |

Summed CPU time of the build's processes (50 Hz sampler, second number only), adopted over stock: memcached +35 %,
Redis +162 %, SQLite +45 %, FFmpeg +23 %, MySQL +19 % (round 3). Wall and CPU differ in a known direction: FFmpeg
and MySQL read higher by CPU (a serial setup shared by all arms dilutes the wall ratio), Redis and SQLite lower (the
adopted build is bounded by its largest unit). Range of the wall figure, adopted over native: +10 % (Chromium, CPU
time of sampled units) and +25 % (MySQL) to +213 % (Redis); over stock +11 % to +197 %.

**With DE in removal form, one window** (14 Oct, the adopted removal lines; native, stock and adopted alternating,
3 builds per arm, medians, 8 jobs; the host was not quiet, load 2.7 to 16, so the status is current):

| app | native | stock TSan | adopted | stock over native | adopted over native | adopted over stock | status |
|---|---|---|---|---|---|---|---|
| memcached | 4.97 s | 6.85 s | 12.08 s | +38 % | +143 % | +76 % | current (whole-program step 4.86 s of the 12.08 s) |
| Redis | 4.97 s | 5.35 s | 15.85 s | +8 % | +219 % | +196 % | current (whole-program step 4.77 s of the 15.85 s) |
| SQLite | 22.88 s | 25.81 s | 39.74 s | +13 % | +74 % | +54 % | current |
| FFmpeg (N1 placed statically) | 70.61 s | 82.81 s | 93.47 s | +17 % | +32 % | +13 % | current |

memcached and Redis are from a second pass at the end of the window (builds agree within 2 % and 5 %); SQLite's and
FFmpeg's spread is at most 8 %. Redis reads 15.85 s here against 14.31 s for the verified form on 12 Oct; the two
were built on different nights, so this is not a measured difference between the forms. MySQL keeps the row above.
Range over the five applications, adopted over native: +25 % (MySQL) to +219 % (Redis); over stock +13 % to
+196 %.
| Chromium (compile CPU seconds over a fixed sample of 400 units, sum of per-unit medians, 3 repetitions) | 1476.9 s | 1459.0 s | 1621.0 s (FE-INL configuration 1480.8 s) | −1.2 % | +9.8 % [7.3, 12.7] (+11.1 % [8.5, 14.7] over stock); FE-INL configuration +0.3 % (+1.5 % over stock) | current (14 Oct, Intel, 4800 compiles, 0 failed; host busy outside the measuring cores; stock compiles faster than native because the native configuration of the browser adds hardening flags that its sanitizer configuration drops; per-unit compile time, not build wall time, link not included) |


| app | configuration | speedup | stock build | optimized build | overhead | status |
|---|---|---|---|---|---|---|
| SQLite | AllOpt (LO-MARK v1) | 1.23× | 24.86 s | 37.53 s | +51 % | final. Generating the spec: 98.7 s on one CPU, once per source version (with it counted into one build: 136.2 s, 5.5× stock); the same analyses without the spec: 30.77 s, +24 % |
| FFmpeg | second line: N1 at every check, `fcn` | 1.29× | 73 s | 136 s | +86 % | final (3 builds per arm) |
| Redis | adopted | 1.49× | 5.37 s | 14.38 s | +168 % | current (static placement under test) |
| FFmpeg | AllOpt (N1 placed statically, `fk6`) | 1.24× | 76.44 s | 89.70 s | +17 % | final (3 builds per arm, 13 Oct) |
| FFmpeg | alternative: N1 placed statically, `fc4` | 1.24× | 72 s | 95.5 s | +33 % | current (2 builds per arm, whole seconds) |
| Redis | alternative: N1 placed statically, `rk4` | 1.39× | 5.25 s | 11.99 s | +128 % | current; not adopted (adopted line 1.51× in the same leg, 14.31 s, +173 %) |
| MySQL | FE-INL | 1.13× | 874.75 s | 1015.13 s | +16 % | current |
| memcached | AllOpt (EVCONF-CHECKED) | 1.81× over upstream stock | 9.32 s | 16.32 s | +75 % | final (3 builds per arm; native 6.85 s: stock +36 % and AllOpt +138 % over native; the whole-program summary step is 6.53 s of the 16.32 s, without it +5 % over stock) |
| Chromium | AllOpt | 1.25× | | | +10 % (CPU time, 400 sampled units) | final |

## 6. Race preservation

Runs that report the site, of runs made; reporting on; stock and the configuration under the same load.

SQLite, LO-MARK v1 (the adopted configuration), 12 Oct, 15 tests, 20 runs per arm, three independent stock arms;
no site lost, no voided run:

| site | stock A | stock B | stock C | LO-MARK v1 (95 % Wilson) |
|---|---|---|---|---|
| `walRestartHdr` (line 68035) | 10 of 20 | 4 of 20 | 10 of 20 | 4 of 20 [0.08, 0.42] |
| `walRestartHdr` (line 68036) | 10 of 20 | 3 of 20 | 5 of 20 | 5 of 20 [0.11, 0.47] |
| `walIndexRecover` (line 67451) | 8 of 20 | 9 of 20 | 6 of 20 | 8 of 20 [0.22, 0.61] |
| `walIndexRecover` (line 67460) | 0 | 1 | 1 | 0 |
| `sqlite3_enable_shared_cache` (line 7164) | 0 | 1 | 1 | 0 |

Status: current (audited 12 Oct). Every rate of LO-MARK is inside the range of the three stock arms; stock against
stock differs as much (one site 3 of 20 in one stock arm and 0 of 20 in the other two). In the session of 9 Oct
(N = 20) LO-MARK was below both stock arms at the three WAL sites (5 against 13 and 6, 2 against 10 and 4, 4 against
10 and 8), so "inside stock's range" holds for 12 Oct only. The three rare sites (0 or 1 of 20) are undetermined,
not shown kept. Seeded control: an unlocked
write to a page buffer is reported by stock in 5 of 5 runs and voids the run in 5 of 5 under LO-MARK; never silent.

Chromium, runs reporting the race, all sessions:

| race | stock | FE-INL configuration | AllOpt |
|---|---|---|---|
| crashpad `WorkerThread::Stop` (Speedometer) | 33 of 33, both together | | 10 of 19 |
| perfetto start-up (`paint`) | 4 of 40 | 4 of 20 | 3 of 10 |
| PartitionAlloc bit-field (Speedometer) | 11 of 21 | 6 of 12 | 2 of 19 |

Status: current. The crashpad runs without a report are eviction in the unmodified runtime.

SQLite, single configurations, 13 Oct, 10 runs per arm (stock | STC | SWMR | LO | EA | DE | DE+Peeling | FE-INL):
`walRestartHdr` 68035: 9 | 7 | 6 | 9 | 6 | 8 | 7 | 8; 68036: 9 | 8 | 6 | 9 | 7 | 6 | 5 | 9; `walIndexRecover` 67451:
3 | 3 | 3 | 2 | 2 | 0 | 3 | 5. DE's 0 of 10 was rechecked in one session of 20 runs: stock 6 of 20, DE 6 of 20
(DE+Peeling 3 of 10); the checks at that site are identical in the three builds. No site lost in any arm. final.

Redis, single configurations: stock reports no race on this workload, so 11 seeded races were run, 10 runs per
arm (stock, STC, SWMR, EA, DE, DE+Peeling, FE-INL): 10 of them reported in 10 of 10 runs in every arm; the
timing-dependent one in 8 of 10 by stock and 6 to 9 of 10 by the arms. What this certifies: in DE and DE+Peeling
every check in the seeded functions is an inline test, so all 11 seeds exercise it; in FE-INL they exercise the
stack attribution; in STC, SWMR and EA the seeded functions are compiled as in stock, so no seed is aimed at those
three (their code changes are not tested by the seeds). current.

Redis, the DynSTC column with quiet threads (new arm of 13 Oct) and the adopted line, both built with the seeded
base spec, 11 seeds × 10 runs each: ten seeds reported in 10 of 10 runs in both; no seed reported less often in the
arm than in the adopted line. 21 of 22 seed functions carry phase entries, so the set is aimed at the guard. On
the timing-dependent seed, runs with neither a race report nor a guard violation: 2 of 10 in the arm, 4 of 10 in
the adopted line (stock reports that seed in 1 of 10). Read in both binaries: the main thread's racing store is never skipped (a
plain check in the arm, an inline test with no single-mode branch in the adopted line), so a skip interval cannot
hide this race; a run without a report is the detector's own miss of the pair, as in stock. The violation is raised
by a signal handler entering on a quiet thread while the main thread skips, not by the racing access; whether that
handler check can miss an entry is covered by the suite's deterministic tests, not by this seed. One stray guard violation per arm in 110
runs. On one seed the arm switches the guard off at a background job (it has no loop peeling, so the start-up
latch sits in the only copy of the loop); the race is reported. current.

memcached, single configurations (13 Oct, non-timed host, 10 runs of stock and of each of STC, SWMR, LO, EA, DE,
DE+Peeling, FE-INL): stock reports 4 sites, each in 10 of 10 runs; every arm reports each of them in 10 of 10; no
site lost. The LO arm shows 12 further sites in one of its 10 runs that stock and the other arms never show (extra
reports, not read further). The EA arm with EVCONF-CHECKED, 10 runs against 10: each of stock's 4 sites in 10 of 10, no void line, owner census 7.84 M marks in every run. final.

memcached with DE in removal form (13 Oct, 5 runs per arm): the adopted line, DE and DE+Peeling report each of
stock's 4 sites in 5 of 5 runs; no void line; owner census 7.84 M marks in every run of the adopted line. With
this, preservation of the removal form is complete for all five applications: no site lost, no seed below stock.
current.

MySQL with DE in removal form (13 Oct, 5 runs per arm on the non-timed host, 108 sites of stock): no site lost.
Kept / undetermined, adopted line 92 / 3, DE 97 / 0, DE+Peeling 93 / 1; no server death, no disk error. At 5
runs a site that stock shows in 1-2 runs and the arm in none is undetermined. current.

FFmpeg with DE in removal form (13 Oct, 9 seeds × 10 runs per arm): the adopted line and the line with N1 at every
check (both with `dynstc_rt=1`), DE and DE+Peeling in both placements report every seed in 10 of 10 runs, as stock.
current.

SQLite with DE in removal form (13 Oct, 15 tests, 10 runs per arm): no site lost in the adopted line, DE or
DE+Peeling against stock. Runs reporting each of stock's three sites, stock | adopted | DE | DE+Peeling: 9 | 8 |
10 | 10; 6 | 8 | 9 | 9; 4 | 3 | 2 | 4. The adopted line: no void line in 10 runs, 19.6-23.6 M held hits per run.
current.

Redis with DE in removal form (13 Oct, 11 seeds × 10 runs per arm, seeded builds of the same lines): the adopted
line with the guard reports ten seeds in 10 of 10 runs, as its verified twin; on the timing-dependent seed, runs
with neither a race nor a violation 2 of 10 (verified twin 4 of 10); the DE and DE+Peeling arms report every seed
in 10 of 10 except the one stock itself reports in 8 of 10 (9 and 8 of 10). No seed that the verified form reports
is lost. current.

FFmpeg with `dynstc_rt=1` only (the options of the candidate adopted line; 13 Oct): the static line, the line with
N1 at every check and the DynSTC arm report each of the 9 seeds in 10 of 10 runs, as stock. That the option was
active rests on the runner's option line (no statistics line is printed without the phase-guard option). current.

FFmpeg, the four new arms of 13 Oct (DynSTC with N1-ST in both placements, DE and DE+Peeling with static placement): every seed in 10 of 10 runs, as stock; seed functions whose checks differ from stock's, of 6: 5 (N1 at every check), 3 (DE+Peeling), 2 (DE), 1 (DynSTC with static placement). current.

FFmpeg, single configurations: stock reports no race, so 8 seeded races were run, 10 runs per arm (stock, STC,
SWMR, LO, EA, DE, DE+Peeling, FE-INL): every seed reported in 10 of 10 runs in every arm. Aimed: DE and DE+Peeling
(inline test on every check in the seeded functions), SWMR and EA (they remove checks inside seed-carrying
functions; not mapped to the seeded lines), FE-INL (stack attribution). Not aimed: STC and LO (seeded functions as
in stock). current.

MySQL, single configurations (STC, SWMR, LO, EA, DE, DE+Peeling), 7 runs per arm on the non-timed host (13 Oct;
5 runs of 12 Oct and 2 more; a further 3 were cut short by a full disk and are not counted), reports keyed by
function pair: 187 sites reported by stock. Per arm, kept with the same lines / kept with other lines /
undetermined (stock in 1-2 of 7 runs, the arm in none): STC 175 / 0 / 12; SWMR 177 / 1 / 9; LO 176 / 0 / 10; EA
172 / 1 / 14; DE 176 / 0 / 9; DE+Peeling 177 / 1 / 8. The rule (stock k of 7, arm 0, (1 − k/7)^7 < 0.05) fires
four times, each with stock at 3 of 7, and none is a lost check: a start-up race on the temporary tablespace in
DE+Peeling, where both functions are instruction-for-instruction identical to stock's in the arm's binary; a pair
in the wait array in DE and in LO, the latter carrying exactly stock's checks; and one pair in DE that the arm
reports with the read and write roles exchanged. No server death other than those caused by the full disk. So at
7 runs the rule also fires on an arm with stock's instrumentation; a hit is settled by reading the binary. The
adopted line's binary keeps both checks of the start-up race as well. current.

Rows of 14 Oct, non-timed host, 10 stock against 10 arm runs, all exit 0, no void; site = function pair; identical
= same line pair, equivalent = same functions at other lines (0 everywhere):

| app | arm | stock pairs | identical | undetermined | lost by the rule | only in the arm |
|---|---|---|---|---|---|---|
| memcached | adopted line, removal form | 11 | 11 | 0 | 0 | 0 |
| memcached | DE, removal form | 11 | 10 | 0 | 1 | 0 |
| memcached | DE+Peeling, removal form | 11 | 11 | 0 | 0 | 0 |
| memcached | DynSTC | 11 | 11 | 0 | 0 | 0 |
| MySQL | DynSTC (stock's binary with the run-time option) | 184 | 179 | 4 | 1 | 5 |
| MySQL | FE-INL | 184 | 183 | 1 | 0 | 7 |
| MySQL | adopted line (verified form) | 184 | 180 | 3 | 1 | 6 |
| SQLite | DynSTC | 4 line pairs | 3 | 1 (3 of 10 against 0 of 10) | 0 | 0 |
| SQLite | adopted line, removal form (13 Oct) | 5 line pairs | 5 | 0 | 0 | 0 |

memcached's writer sites: 4 of 4 in 10 of 10 runs in every arm. The rule fires three times and none is a removed
check: (1) MySQL adopted line, read and write of a monitor counter in `monitor_set_max_value`: stock 5 of 10, arm 0
of 10; the arm's binary keeps both checks as unconditional runtime calls, as stock; the same line in removal form
reported it in 3 of 5 runs, FE-INL and DynSTC in 3 of 10; (2) MySQL DynSTC, a write/write pair in the binlog ticket
manager: stock 3 of 10, arm 0 of 10, and the arm is stock's own binary; (3) memcached DE, a read of the clock in
`lru_pull_tail` against its writer: stock 3 of 10, arm 0 of 10; the check is in the arm's binary, the sibling
reader is reported in 7 of 10 (stock 3 of 10), the writer site in 10 of 10. With about 400 pair-arm comparisons at
the 5 % rule, hits on unchanged checks are expected; each is settled by reading the binary. Extension of 14 Oct,
10 more runs per arm: both MySQL pairs are kept: (1) arm 2 of 20, stock 9 of 20 (kept at a lower rate, both
checks present); (2) arm 2 of 20, stock 3 of 20. memcached's pair: extension running. Chromium's PartitionAlloc race (stock 11 of 21, AllOpt 2 of 19): the same race in
every build; AllOpt's library keeps both checks (the read through the inline test's miss path, the write
unconditional), so no check is removed; the logs do not separate timing from shadow eviction. current.

The four analyses without DE and without run-time-checked parts (14 Oct, seeded builds, 10 runs per arm): Redis,
ten seeds in 10 of 10 runs and the timing-dependent one in 8 of 10, as its seeded stock (8 of 10); FFmpeg, all 9
seeds in 10 of 10, as its seeded stock. No seeded race lost. memcached and SQLite: running. current.

Other applications: 10 stock against 10 optimized runs each, no site lost (memcached 4 sites, MySQL 232-236;
Redis and FFmpeg by seeded races). Per-site rates: pending (final legs' binaries).

## 7. Regression suite

compiler-rt's TSan tests with the flags of the Chromium configurations (391 tests run): all pass. 
Adopted lines (13 Oct; the tests of each root's own commit, flags and run-time options of the line; flags bound to
one program — specs, candidate lists, summaries — left out; 90 tests unsupported on the host in every row):

| app | line | run | passed | expected failures | failed | status |
|---|---|---|---|---|---|---|
| MySQL | FE-INL line | 392 | 391 | 1 | 0 | final |
| FFmpeg | stock and the six analyses | 391 | 391 | 0 | 0 | final |
| FFmpeg | AllOpt (N1 placed statically), with its options | 391 | 379 | 0 | 12 | current |
| FFmpeg | N1 at every check, with its options | 391 | 382 | 0 | 9 | current |
| SQLite | AllOpt without its spec flags | 445 | 424 | 1 | 20 | current |
| memcached | AllOpt without spec and summaries | 422 | 402 | 1 | 19 | current |
| Redis | AllOpt without the phase flags | 411 | 408 | 1 | 2 | current |

No failure read is a missing race report. 19 of SQLite's, 19 of memcached's and 7 of FFmpeg's failures are one
defect: a unit compiled without optimization under verified DE or the inline hit test does not link on these roots
(a thread-local placeholder is referenced before its first use is set up). The fix exists (decided 12 Oct) but is
in none of the measured roots; the applications are built at -O2 and are not affected. The remaining failures are
tests that check the emitted form of stock instrumentation, control twins whose expected loss does not occur, one
pinned frame, and one report added by the phase-guard option.

Single configurations as built (13 Oct), failed tests per application (run: SQLite 445, memcached 422, Redis 411,
FFmpeg and MySQL 392):

| configuration | SQLite | memcached | Redis | FFmpeg | MySQL |
|---|---|---|---|---|---|
| STC, SWMR, LO, EA | 0 | 0 | 0 | 0 | 0 |
| FE-INL | 1 (a form check) | 0 | 0 | 0 | 0 |
| DE, DE+Peeling, verified form | 12 | 11 | 3 | 1 (static placement: 10) | 11 |
| DE, DE+Peeling, removal form | 0 | 0 | 3 (with the inline test) | 1-11 (with the inline test) | 0 |
| adopted line, removal form | 0 | 0 | 2 | 12 | 0 |
| the four analyses without DE and without run-time-checked parts (14 Oct) | 0 (445 run, 444 passed, 1 expected failure) | 0 (422, 421, 1) | 0 (411, 410, 1) | 0 (392, 391, 1) | not built |

No failure in any row is a missing race report. Every nonzero cell is the link defect of the inline test on
unoptimized code (fix exists, in no measured root) plus checks of the emitted form written for other flags. In
the removal form of DE the inline test is not emitted, so SQLite's, memcached's and MySQL's rows are clean. final.

## 8. Thread-count sweep

memcached (13 Oct, AMD; server threads = `-t`, on the final leg's pinning of 8 logical CPUs; client, workload and
request count unchanged; 2 offsets × N = 2, 36 of 36 timed runs valid):

| server threads | AllOpt over stock | stock, ops/s | AllOpt, ops/s | status |
|---|---|---|---|---|
| 8 | 1.90× [1.89, 1.90] | 216k | 410k | final (A/A 1.000 [0.998, 1.002]) |
| 6 | 1.97× [1.95, 1.98] | 195k | 383k | final |
| 4 | 2.11× [2.09, 2.12] | 161k | 340k | final (one of two passes disturbed and retired; the clean pass 2.12× [2.11, 2.12]) |
| 2 | 2.23× [2.19, 2.27] | 85k | 190k | current (N = 1 per offset; every cell ran with the housekeeping CPUs busy) |

The gain grows as the thread count falls, each step outside the previous interval. Base: the root's own stock;
upstream stock over it at 8 threads 1.046 [1.040, 1.051]. Peak over all points: 2.23× (flagged) or 2.11× (clean).

Second sweep with the adopted line in removal form (14 Oct; points at 8, 6 and 4 threads follow). 2 server threads,
2 offsets × N = 2, 12 of 12 valid, 0 void: **2.27× [2.25, 2.29]** over stock (offsets 2.283 / 2.251); stock 85.7k
ops/s, AllOpt 194.3k; server CPU per run 1166 s and 510 s. Read under a declared co-load on the housekeeping CPUs
(0.07-0.26 per cell, 7 of 13 cells below 0.10, none above 0.40). Peak over all points: 2.27×. final.

| server threads | AllOpt (removal) over stock | EA with the run-time ownership check merged in, over stock | AllOpt over that EA | status |
|---|---|---|---|---|
| 8 | 1.89× (offsets 1.875 / 1.904) | 1.85× (1.836 / 1.863) | 1.02 | current (2 offsets × N = 1; second repetition running) |
| 6 | 1.96× (1.971 / 1.955) | 1.86× (1.898 / 1.831) | 1.05 | current |
| 4 | 2.10× (2.089 / 2.110) | 2.05× (2.021 / 2.078) | 1.02 | current |
| 2 | 2.27× [2.25, 2.29] | queued (plain EA without the merge: 1.00) | | final for AllOpt |

14 Oct 07:05-08:27, 18 of 18 valid, 0 void, declared co-load 0.02-0.20. Server CPU per run at 8 / 6 / 4 threads:
stock 1851 / 1601 / 1237 s, AllOpt 921 / 775 / 573 s. The gain rises as the server threads fall and the merged EA
configuration carries it at every point; the full line adds 2-5 %.

SQLite (13 Oct, on walthread1, the submitted paper's sweep workload, since stress2 has a fixed thread set; 20 s
per run, 4 runs per arm, 32 of 32 valid): AllOpt over stock 1.00 [0.99, 1.00] at 16 threads, 1.01 [1.00, 1.03] at
8, 1.00 [0.97, 1.03] at 4, 1.01 [0.98, 1.03] at 2. The run-time census reads 0 marks and 0 held hits in every run:
walthread1 never reaches the objects LO-MARK covers (stress2: about 1 M mark calls and 10-14 M held hits per run).
So this sweep shows only that the adopted line costs nothing where it does not apply; SQLite's gain has no
thread-count series. final.
FFmpeg (13 Oct, AMD; `-threads` on the final leg's protocol, composite of four transcodes over stock; 2 offsets ×
N = 2, 64 of 64 valid; intervals about ±0.015):

| threads | static line, `dynstc_rt=1` only | static line, both options | N1 at every check, both options | h264 with `phase_guard=1` | h264 with `dynstc_rt=1` only | status |
|---|---|---|---|---|---|---|
| 16 | 1.25× | 1.18× | 1.22× | 0.80 | 1.00 | final |
| 8 | 1.26× | 1.22× | 1.27× | 0.90 | 1.01 | final |
| 4 | 1.28× | 1.25× | 1.30× | 0.95 | 1.00 | final |
| 2 | 1.29× | 1.25× | 1.29× | 0.95 | 1.02 | final |

The gain (the stream copy, 2.4-2.6×) does not depend on the thread count. What scales with threads is the cost of
`phase_guard=1` on h264: 5 % at 2-4 threads, 10 % at 8, 21 % at 16; with `dynstc_rt=1` alone the line is flat.

FFmpeg, the sweep of record with the adopted line (DE in removal form, static placement, `dynstc_rt=1` only) and
the DynSTC arm alone (stock instrumentation plus the run-time test), 14 Oct 04:02-05:34, AMD, 2 offsets × N = 2,
48 of 48 valid:

| threads | adopted line (AllOpt) over stock | DynSTC alone over stock | adopted over DynSTC | status |
|---|---|---|---|---|
| 16 | 1.27× [1.25, 1.28] | 1.27× [1.25, 1.29] | 1.00 [0.99, 1.01] | final |
| 8 | 1.27× [1.25, 1.28] | 1.28× [1.27, 1.29] | 0.99 [0.98, 1.00] | final |
| 4 | 1.28× [1.27, 1.28] | 1.28× [1.27, 1.28] | 1.00 [0.99, 1.00] | final |
| 2 | 1.28× [1.27, 1.28] | 1.29× [1.28, 1.30] | 0.99 [0.99, 1.00] | final |

Per transcode at 4 threads, adopted / DynSTC: copy 2.52 / 2.55, h264 1.00 / 1.00, h265 1.00 / 1.00, mjpeg 1.05 /
1.05. The adopted line does not depend on the encoder thread count, and the DynSTC arm alone equals it at every
point: on FFmpeg the gain is the run-time single-thread test on the stream copy; the static analyses and DE add
nothing measurable on top of it.
