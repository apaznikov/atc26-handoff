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
| SQLite | | | | | | | | | 1.19× | current |
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
Speedometer 3 on Intel: pending (AMD, Score over stock, N = 3: FE-INL configuration 1.082, AllOpt 1.006, A/A 1.009).
Two of about 26 Speedometer runs of AllOpt died on a debug check of V8's garbage collector; no other build did; the
compiled code points to a timing-dependent condition in V8, confirmation on stock pending.

## 3. Slowdown over native, memory

Pending: no native arm has been measured yet (final record legs; Chromium needs a native build).

| app | stock / native | AllOpt / native | memory overhead reduction | status |
|---|---|---|---|---|
| | | | | pending |

Chromium, largest process's peak resident memory over stock: FE-INL configuration 1.09×, AllOpt 1.77× (mapped
code; allocated memory unchanged). current.

## 4. Instrumentation reduction and code size

Pending (static and dynamic reduction by the protocol's definitions, from the final legs' binaries).
Chromium code size over stock: FE-INL configuration 1.18×, AllOpt 3.0×. current.

## 5. Compile time

Over the stock TSan build; fresh trees, medians of 3, wall seconds. Overhead against the native build: pending.

| app | configuration | speedup | stock build | optimized build | overhead | status |
|---|---|---|---|---|---|---|
| SQLite | LO-OBJ, generated spec | 1.19× | 33.27 s | 43.28 s | +30 % | current (LO-MARK's to be taken) |
| FFmpeg | N1 at every check | 1.285× | 76.10 s | 149.88 s | +97 % | current (static placement under test) |
| Redis | adopted | 1.49× | 5.37 s | 14.38 s | +168 % | current (static placement under test) |
| MySQL | FE-INL | 1.13× | 874.75 s | 1015.13 s | +16 % | current |
| memcached | adopted | 1.80× | | | not measured | pending |
| Chromium | AllOpt | 1.25× | | | +10 % (CPU time, 400 sampled units) | final |

## 6. Race preservation

Runs that report the site, of runs made; reporting on; stock and the configuration under the same load.

SQLite, LO-MARK v1 (the candidate for the LO column), 12 Oct, 15 tests, no site lost, 0 voided runs of 20:

| site | stock | LO-MARK v1 |
|---|---|---|
| `walRestartHdr` (line 68035) | 8 of 10 | 4 of 10 |
| `walRestartHdr` (line 68036) | 6 of 10 | 2 of 10 |
| `walIndexRecover` (line 67451) | 3 of 10 | 4 of 10 |

Status: current. The lower rate at `walRestartHdr` is under test against stock's own spread (N = 20 per arm). The
accesses there are instrumented identically in both builds. Seeded control: an unlocked write to a page buffer is
reported by stock in 5 of 5 runs and voids the run in 5 of 5 under LO-MARK; never silent.

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
