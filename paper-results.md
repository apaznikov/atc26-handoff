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
| Chromium 127.0.6533.119 | blink_perf (two story samples), Speedometer 3 | Intel | `tsan-cr-8345a0396fa5` | component build, DCHECKs on |

## 2. Speedup over stock TSan

Applications. Columns are the paper's configurations with the new optimizations merged in; AllOpt is the adopted
configuration. All cells but AllOpt wait for the final record legs (13-14 Oct).

| app | STC | SWMR | LO | EA | DE | DE+Peeling | DynSTC | FE-INL | AllOpt | status |
|---|---|---|---|---|---|---|---|---|---|---|
| SQLite | | | | | | | | | 1.28× [1.23, 1.32] | current (stress2 only, AMD; quiet leg 12 Oct, 4 offsets × N = 4, A/A 1.013; LO-MARK v1; the assertion-derived spec reads 1.26× in the same runs; the same trees read 1.11× on Intel) |
| FFmpeg | | | | | | | | | 1.29× | current |
| Redis | | | | | | | | | 1.54× | current |
| MySQL | | | | | | | | | 1.13× | current |
| memcached | | | | | | | | | 1.80× | current |

Chromium, Intel, N = 3 repetitions, speedup over stock:

| configuration | first sample (49 stories) | hold-out sample (48 stories) | status |
|---|---|---|---|
| the four analyses (STC, SWMR, LO, EA) | 1.005 | — | final |
| + DE | 1.012 | — | final |
| + peeling | 0.993 | — | final |
| the four analyses + FE-INL + MEMINTR | 1.104 | 1.101 | final |
| AllOpt (+ N1) | 1.286 | 1.248 | final |

Noise floor: a layout twin of the FE-INL configuration reads 1.013 over it. AllOpt on the AMD host: 1.081 and 1.017.

Speedometer 3, Score over stock, N = 3 repetitions (range):

| configuration | Intel | AMD | status |
|---|---|---|---|
| the four analyses + FE-INL + MEMINTR | 1.112 (1.082–1.158) | 1.082 (1.049–1.147) | final |
| AllOpt (+ N1) | 1.225 (1.201, 1.250; N = 2, one run failed) | 1.006 (0.983–1.048) | final |
| A/A | 1.012 (0.994–1.044) | 1.009 (0.972–1.042) | final |

On Intel one repetition of the FE-INL configuration ran on a disturbed host; without it the figure is 1.090 (N = 2).
Two of about 26 Speedometer runs of AllOpt died on a debug check of V8's garbage collector; no other build did; the
compiled code points to a timing-dependent condition in V8. Confirmed: with V8's forced compaction the stock TSan
build dies on the same check (1 of its first 2 runs). Whether AllOpt hits it more often than stock in normal runs
(3 of about 27 against 0) is being counted.

## 3. Slowdown over native, memory

Pending: no native arm has been measured yet (final record legs; Chromium needs a native build).

| app | stock / native | AllOpt / native | memory overhead reduction | status |
|---|---|---|---|---|
| | | | | pending |

Chromium, largest process's peak resident memory over stock: FE-INL configuration 1.06×, AllOpt 1.54× on Intel
(Speedometer); 1.09× and 1.77× on AMD (mapped code; allocated memory unchanged). current.

## 4. Instrumentation reduction and code size

Pending (static and dynamic reduction by the protocol's definitions, from the final legs' binaries).
Chromium code size over stock: FE-INL configuration 1.18×, AllOpt 3.0×. current.

## 5. Compile time

Over the stock TSan build; fresh trees, medians of 3, wall seconds. Overhead against the native build: pending.

| app | configuration | speedup | stock build | optimized build | overhead | status |
|---|---|---|---|---|---|---|
| SQLite | LO-OBJ, generated spec | 1.19× | 33.27 s | 43.28 s | +30 % | current (not the adopted configuration; LO-MARK's compile time is not measured) |
| FFmpeg | N1 at every check | 1.285× | 76.10 s | 149.88 s | +97 % | current (static placement under test) |
| Redis | adopted | 1.49× | 5.37 s | 14.38 s | +168 % | current (static placement under test) |
| FFmpeg | candidate: N1 placed statically, `fk6` | under test | 72 s | 85 s | +18 % | current (2 builds per arm, whole seconds) |
| FFmpeg | candidate: N1 placed statically, `fc4` | under test | 72 s | 95.5 s | +33 % | current (2 builds per arm, whole seconds) |
| Redis | alternative: N1 placed statically, `rk4` | 1.39× | 5.25 s | 11.99 s | +128 % | current; not adopted (adopted line 1.51× in the same leg, 14.31 s, +173 %) |
| MySQL | FE-INL | 1.13× | 874.75 s | 1015.13 s | +16 % | current |
| memcached | adopted | 1.80× | | | not measured | pending |
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

Other applications: 10 stock against 10 optimized runs each, no site lost (memcached 4 sites, MySQL 232-236;
Redis and FFmpeg by seeded races). Per-site rates: pending (final legs' binaries).

## 7. Regression suite

compiler-rt's TSan tests with the flags of the Chromium configurations (391 tests run): all pass. Per application
configuration: pending.

## 8. Thread-count sweep

Pending (after the record legs; SQLite, memcached, FFmpeg).
