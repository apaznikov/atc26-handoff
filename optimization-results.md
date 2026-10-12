# TSan instrumentation optimizations: results

State: 12 Oct 2026. This is the short form. The full ledger with every leg, control and dead end is the previous
version of this file, commit `9126669`. Soundness, premises and audits: [`soundness-fixes.md`](soundness-fixes.md).

Contents: [1. Summary](#summary) · [2. Chromium](#chromium) · [3. Single optimizations](#single) ·
[4. No gain](#nogain) · [5. Compile time](#compile) · [6. What each optimization does](#explain) ·
[7. Method](#method)

<a name="summary"></a>
## 1. Summary

Speedup = stock TSan's run time over the optimized build's. Every figure is sound: no race that stock TSan reports is
lost, under the premises of `soundness-fixes.md` §6. Compile time is over stock TSan's build.

| app | workload | submitted paper | no annotations, static | no annotations, checked at run time | generated spec or annotations | compile time |
|---|---|---|---|---|---|---|
| SQLite | threadtest3 stress2 | 1.71× | ≈ 1.0× ([the paper's analyses](#paper)) | 1.26× ([LO-MARK](#lo-mark); measured, not yet a claim) | **1.19×** ([LO-OBJ-G](#lo-obj-g), generated spec); 1.24× with run-time owner checks; 1.25× hand-written spec | +28 % |
| FFmpeg | four transcodes of one film | 1.57× | — | **1.29×** ([DynSTC-RT](#dynstc-rt) + [N1](#n1) + [N1-ST](#n1-st)); **1.23×** with the hit test placed by a [static rule](#n1-place) | — | +97 %; static rule +36 % |
| Redis | 7 data-heavy commands, 8 I/O threads | 1.45× | 1.10× ([FE-INL](#fe-inl) + [N1](#n1)) | **1.54×** (+ [quiet threads](#quiet); a fresh build on 12 Oct read 1.49×) | — | +168 % |
| MySQL | Release, sysbench insert/update/delete, 24 connections | 1.16× | **1.13×** ([FE-INL](#fe-inl)) | — | — | +16 % |
| memcached | pipelined 32-key gets, 190-byte keys | 1.07× | **1.02×** (analyses + [EA-CONTENTS](#ea-contents) + [SWMR-ROOTS](#swmr-roots) + [FE-INL](#fe-inl) + [MEMINTR](#memintr)) | **1.80×** ([EVCONF-CHECKED](#evconf-checked)) | 1.81× ([EVCONF](#evconf), 7 annotated fields) | +11 %; with EVCONF +35 % |
| Chromium | blink_perf, 49 + 48 stories | 1.39× | Intel: **1.10×** ([CR1](#cr1)), **1.16×** ([S56](#n1-place)), **1.25×** ([CR1+N1](#cr1)); AMD: 1.06× / 1.05× / 1.02× (hold-out sample) | — | — | CR1 +1 %; CR1+N1 +10 % |

- Hosts: AMD = 2 × EPYC 9115 (all rows except Chromium's Intel figures); Intel = Xeon w9-3495X.
- SQLite's figure is completed statements per 60 s of a lock-contended mix dominated by integrity checks (33-36 %) and
  schema statements; the write loops are 7-9 % of it. It is a throughput ratio, not a speedup of write work.
- SQLite's row is being re-recorded without dominance elimination and peeling (same speed in a screening leg, 1.185
  against 1.187; compile time +20 % against +37 % by CPU time).
- FFmpeg: of the 1.29×, 1.085× is the runtime option of DynSTC-RT alone; the instrumentation gives 1.18×.
- memcached's 1.80× and MySQL's 1.13× are over upstream LLVM's TSan; ours and upstream's stock differ by -4 to +1.5 %.
- The submitted paper's figures do not reproduce on the camera-ready compiler with the soundness fixes: the paper's
  analyses alone give about 1.0× on SQLite, memcached and Chromium.

<a name="chromium"></a>
## 2. Chromium

Chromium 127.0.6533.119, component build (486 libraries) with DCHECKs on, compiler `tsan-cr-8345a0396fa5`
(`tsan-cr2-429bd8cf76b9` for S56). First sample: 49 Blink stories of six suites, used during development. Hold-out
sample: 48 other stories, fixed before they ran. Speedup over stock TSan, N=3 unless noted.

| configuration | Intel, first | Intel, hold-out | AMD, first | AMD, hold-out |
|---|---|---|---|---|
| [Sound](#cr1) (the four analyses) | 1.006 (N=2) | — | 0.995 | — |
| [AllOpt](#cr1) (+ dominance elimination, peeling) | 0.994 (N=2) | — | 1.011 | — |
| [CR1](#cr1) | 1.104 | 1.101 | 1.068 | 1.057 |
| CR1 without the library-facts table | 1.102 (N=2) | 1.106 (N=2) | 1.070 | — |
| [S56](#n1-place) | 1.186 | 1.157 | 1.086 | 1.054 |
| [CR1+N1](#cr1) | 1.286 | 1.248 | 1.081 | 1.017 |

- **The paper's analyses give about 0 on Chromium** on both hosts. The gain comes from the form of the
  instrumentation (CR1's inlined entry/exit and the inline hit test), which is not in the reviewed paper.
- **CR1 holds on unseen stories**: 1.10× on Intel, 1.06× on AMD.
- **The inline hit test pays only on Intel**: +13 to +17 % over CR1. On AMD's hold-out it is 4 % below CR1 (four
  readings of the same sign). S56 equals CR1 on AMD.
- **Why the hosts differ is not established.** Excluded: thread count (Intel pinned to 16 threads reads the same),
  the container, fonts and paths (AMD on the host reproduces the container within 0.006). Counters show the hit test
  removing cycles in the runtime and adding about as many outside it on both hosts.
- **Peeling** costs 1.8 % on Intel and gains 1.1 % on AMD; each sign is stable on its host.
- **The library-facts table is not needed**: CR1 without it reads the same everywhere.
- **Whole `paint` suite** (AMD, N=2): CR1 1.062, CR1+N1 1.160, S56 1.169; without the multicol stories 1.048, 1.030,
  1.044.
- **Code and memory**: code 1.18× stock's for CR1 and 3.0× for CR1+N1. Largest process's peak RSS: CR1 1.09×, S56
  1.27×, CR1+N1 1.77× (mapped code; allocated memory unchanged).
- **Race preservation** (10 runs of the Blink stories per build): no build loses a race that stock reports. The
  crashpad race is reported in 10 of 19 CR1+N1 runs against 27 of 27 for stock and CR1; this is shadow-cell eviction.
  The perfetto race is reported by the optimized builds (CR1 4 of 20, CR1+N1 3 of 10) and never by stock (0 of 30).
- **compiler-rt TSan tests with the flags** (391 tests run): CR1, CR1+N1 and S56 pass all. The hit-test flag alone
  fails one test at -O0 (a link error from an unused declaration; fix approved 12 Oct).
- **Open**: one CR1+N1 Speedometer story failed in 19 runs (a renderer disappeared, no sanitizer output; it did not
  recur). Speedometer 3 at three repetitions is running on both hosts.

<a name="single"></a>
## 3. Single optimizations

Change of speed in percent from adding one optimization. Rows down to MEMINTR: over the paper's analyses with every
soundness fix. Rows below: on top of their app's configuration.

Kind: S static · R checked at run time, no annotation · G spec generated from the program's assertions · A
hand-written annotation (A+R: guarded at run time). `f` = measured on Intel, otherwise AMD. **Bold** = gain beyond
the A/A (the same binary run twice); "—" = not measured.

| optimization | kind | SQLite | memcached | Redis | MySQL | FFmpeg |
|---|---|---|---|---|---|---|
| [DynSTC-RT](#dynstc-rt) | R | +0.4 | −0.3 | `f` −2.2 | +0.1 | `f` **+12.0** |
| [DynSTC](#dynstc) (the paper's guard) | R | — | −0.5 | −5.5 | −0.7 | `f` **+10.1** |
| [N1](#n1) | S | −0.8 | +0.3 | `f` +2.5 | −1.7 | `f` **+5.0** |
| [N1-ST](#n1-st) (over N1 + DynSTC-RT) | R | +0.2 | −0.2 | `f` −1.4 | −1.8 | `f` **+16.9** |
| [N1-L](#n1-place) | S | +0.3 | −0.9 | `f` −2.5 | −0.1 | `f` **+3.0** |
| [FE-INL](#fe-inl) | S | −0.7 | +0.8 | `f` **+4.8** | **+3.6** | `f` 0.0 |
| [FE-INL + VWIDE-loops](#vwide) | R | +0.1 | +0.6 | `f` **+7.2** | **+4.6** | `f` +1.8 |
| [FE-SINK](#fe-sink) (nothing on top of FE-INL) | S | +1.1 | −1.7 | `f` **+4.1** | **+2.8** | `f` −1.2 |
| [MEMINTR](#memintr) | S | +2.5 | +0.9 | `f` −2.5 | +0.3 | `f` 0.0 |
| [EA-CONTENTS](#ea-contents) | S | — | **+1.4** | — | — | — |
| [SWMR-ROOTS](#swmr-roots) | S | — | **+0.9** | — | — | — |
| [EVCONF](#evconf) (with the two rows above) | A+R | — | **+31.1** | — | — | — |
| [EVCONF-RANGES](#evconf) | A+R | — | **+18.5** | — | — | — |
| [EVCONF-ARGS](#evconf) | A+R | — | **+7.3** | — | — | — |
| [EVCONF-INTERCEPT](#evconf) | A+R | — | **+7.5** | — | — | — |
| [EVCONF-CHECKED](#evconf-checked) (over upstream stock) | R | — | **+80** | — | — | — |
| [Quiet threads](#quiet) | R | — | — | **+31.9** | — | — |
| [LO-OBJ-G](#lo-obj-g), generated spec (over stock) | G+R | **+15.7** | — | — | — | — |
| [LO-OBJ-G](#lo-obj-g), hand-written spec (over stock) | A+R | **+25.0** | — | — | — | — |

<a name="nogain"></a>
## 4. No gain

Tried and closed; one row per family. Shares are of executed checks. Details of each: commit `9126669`, tables 3 and 6.

| family | what was tried | result |
|---|---|---|
| Dominance elimination, further rules | covered checks deleted instead of [verified](#de); one range check per loop; adjacent fields merged; a cover on every path; covers across loop iterations; post-dominance with proven termination; stronger alias analysis | within ±1 % on every app (Redis −3.8 % for deletion); each rule reaches ≤ 2 % of checks |
| Loop guard | a loop-invariant access checked once per sync-free stretch | Redis +2 to +3 %, others ≈ 0; rejected as imprecise |
| Escape analysis, finer | whole-program summaries; per-call-site parameter facts; slots and field chains; fresh return values; thread-local roots | each < 2-3 % of checks; the mass is heap reachable from shared structures, which no static rule reaches |
| STC, SWMR, LO, finer | closed-world facts; join-aware STC; thread roles; publish-once; written-before-publication; POSIX calls transparent to LO | each < 3 % of checks; two variants lost races and were withdrawn |
| [N1](#n1) variants | compact or batched tests; shared thread-state load; test for relaxed atomics; out-of-line entries saving fewer registers | ±1 %, or −2 to −4 %; the batched form loses races |
| [N1](#n1) placement for speed | profile hot list; static frequency; loops only | none beats the full version in speed; they trade speed for compile time ([section 5](#compile)) |
| [FE-INL](#fe-inl) variants | lazy entry; only hot functions; one exit; no entry/exit in functions without accesses | ±1 %, MySQL −5 % for one exit; the last changes report stacks and was not taken |
| N1, FE-INL, MEMINTR on top of SQLite's LO-OBJ-G | — | FE-INL +4.5 %, MEMINTR +3.5 % inside a wide A/A; N1 −18.5 % |
| More optimization before instrumentation | LTO; Attributor; extra passes | nothing; the Attributor miscompiles |
| Ownership, further | owner tags without a record; memcached's flag globals; MySQL connection fields; SQLite private connections; pager and WAL under the b-tree's mutex; quiet mode on the other four apps | unsound, or needs a premise that was declined, or ≤ 2.4 % of checks |
| The old "same location" rule | fields of one struct or elements of one array cover each other | unsound: loses a memcached race stock reports (ceiling +7 to +31 %) |
| Runtime-only changes | covering records count as hits; cheaper range checks; cheaper acquires; deadlock-detector cost; allocation bookkeeping | separate track, not in the paper: [`runtime/ideas.md`](runtime/ideas.md) |

Where the analyses stay conservative (share of executed checks they cannot remove):

| what cannot be proven | SQLite | memcached | Redis | MySQL | FFmpeg |
|---|---|---|---|---|---|
| DE: same address twice, no acquire between, both kept | 3.4 % | 0.04 % | 3.4 % | 5.5 % | 5.9 % |
| DE: neither check dominates the other | 4.4 % | 0.08 % | 5.8 % | 1.0 % | 8.6 % |
| EA: memory that is in fact one-thread or consistently ordered | 72 % | 55 % | 63 % | 49 % | 70 % |
| of it: heap reachable from shared structures | 79 % | 34 % | 42 % | — | 7.5 % |

<a name="compile"></a>
## 5. Compile time

Fresh trees, no ccache, 8 jobs (MySQL 6), stock and the line alternating, medians of 3, wall seconds. For Redis and
memcached the line includes the whole-program summary step.

| app | configuration | stock | line | overhead |
|---|---|---|---|---|
| memcached | no annotations | 7.25 s | 8.03 s | +11 % |
| memcached | with EVCONF | 6.47 s | 8.71 s | +35 % |
| Redis | record (summary generator with kept inputs and parallel analyses, adopted 12 Oct; same binary) | 5.37 s | 14.38 s | **+168 %** |
| SQLite | LO-OBJ-G, generated spec | 33.23 s | 42.57 s | +28 % |
| FFmpeg | hit test everywhere (1.285×) | 76.10 s | 149.88 s | +97 % |
| FFmpeg | hit test by static rule s1 (1.241×) | 76.10 s | 110.77 s | +46 % |
| FFmpeg | hit test by static rule s4 (1.227×) | 76.10 s | 103.53 s | +36 % |
| MySQL | Release, FE-INL | 874.75 s | 1015.13 s | +16 % |
| Chromium | CR1 (400 sampled units, CPU time) | — | — | +1.1 % (1.004-1.018) |
| Chromium | CR1+N1 (same sample) | — | — | +10.4 % (1.076-1.142) |

- **The cost is the [inline hit test](#n1)**: about 25 machine instructions and 133-140 bytes per check. Redis by CPU
  time over stock: analyses +1.6 %, + elimination and peeling +17 %, + inlined entry/exit +24 %, + hit test +157 %.
  The pass itself is 1 % of the added time; the rest is the back end working over the larger code.
- **No change of the pass lowers it**: prototypes of a smaller test gave at most 1-2 % (an assembly form −10 % with
  1.5-1.9 % more code). The one design that would is a back-end pseudo-instruction, recorded as an idea in
  [`runtime/ideas.md`](runtime/ideas.md) and not built.
- **Fewer tests cost speed on Redis** (AMD, over stock): record 1.49×; [static rules](#n1-place) s1 1.34× (+73 % CPU
  time), k2 1.32× (+46 %), s4 1.29× (+42 %); no hit test 1.28× (+26 %).
- **FFmpeg**: no static rule separates the hot checks from the bulk (s1 keeps 48 % of the hot locations for 23 % of
  the code growth; function-size caps keep 2-13 %). The whole speed difference is in the stream-copy transcode.
- **SQLite** by CPU time: record +37 %; without peeling +27 %; without elimination and peeling +20 %, at the same
  speed. The owner checks and the analyses are those 20 points.
- **Redis's summary step** was 7.9 s of the 17.8 s (+230 %). Not rerunning `make -n` and the dependency build, and
  running the whole-program analyses in parallel, brings it to 4.6 s. Summaries and server code are identical.
- Being measured (12 Oct): where the hit-test hits concentrate on Redis and FFmpeg, to place the test by a profile
  list or per unit.

<a name="explain"></a>
## 6. What each optimization does

The examples are schematic C. `__tsan_read4(p)` / `__tsan_write4(p)` stand for TSan's check of a 4-byte access; real
entry names and the inlined code differ in detail.

<a name="paper"></a>
### The paper's analyses

<a name="stc"></a>
**STC, single-threaded context.** Code that runs only before the program creates its first thread needs no checks.

```c
int main(void) {
    g_config.n = 4;                 // before: __tsan_write4(&g_config.n);  after: nothing
    pthread_create(&t, 0, worker, 0);
}
```

<a name="swmr"></a>
**SWMR, single writer / multiple readers.** If every write to a variable happens in single-threaded mode, its reads
need no checks: two reads cannot race.

```c
void *worker(void *a) {
    int n = g_config.n;             // before: __tsan_read4(&g_config.n);   after: nothing
}                                   // (g_config.n is written only in main, before the threads)
```

<a name="lo"></a>
**LO, lock ownership.** If every access to a variable that can run in multi-threaded mode holds one common lock, none
of them is checked.

```c
pthread_mutex_lock(&lock);
g_counter++;                        // before: __tsan_write4(&g_counter);   after: nothing
pthread_mutex_unlock(&lock);        // (every other access to g_counter also holds `lock`)
```

<a name="ea"></a>
**EA, escape analysis.** An object whose address never leaves its thread is not checked.

```c
void handle(void) {
    req_t req;                      // never stored to a global, a shared object or another thread
    req.id = next_id();             // before: __tsan_write4(&req.id);      after: nothing
}
```

<a name="de"></a>
**DE, dominance elimination.** A second access to the same address needs no new check if a checked access dominates
it and nothing between them can synchronize. In the builds of record the covered check is not deleted but *verified*:
it becomes a short inline test that the earlier record is still in the shadow cell, and calls the runtime only if not.

```c
x = p->len;                         // __tsan_read4(&p->len)
y = p->len + 1;                     // before: __tsan_read4(&p->len);  after: covered by the check above
```

<a name="peel"></a>
**Peeling.** The first iteration of a loop is split off so that its check dominates, and so covers, the same access in
the remaining iterations.

```c
for (i = 0; i < n; i++) s += *p;    // before: __tsan_read4(p) in every iteration
// after:
if (n > 0) { __tsan_read4(p); s += *p; }
for (i = 1; i < n; i++) s += *p;    // covered
```

<a name="dynstc"></a>
**DynSTC.** A guard compiled around the checks skips them while the program has one thread.

```c
if (__tsan_multithreaded) __tsan_read4(p);   // before: __tsan_read4(p) unconditionally
x = *p;
```

### The form of the instrumentation

<a name="fe-inl"></a>
**FE-INL, inlined function entry and exit.** TSan keeps a shadow call stack for its reports and calls the runtime at
every function entry and exit. FE-INL does the push and the pop in place.

```c
int get(obj *o) {
    __tsan_func_entry(pc);          // after: *thr->shadow_stack_pos++ = pc;   (a few instructions)
    ...
    __tsan_func_exit();             // after: thr->shadow_stack_pos--;
}
```

<a name="fe-sink"></a>
**FE-SINK.** The entry is moved down to the first point that needs the frame, so early returns record nothing.

```c
int get(obj *o) {
    if (!o) return 0;               // after: no entry/exit on this path
    __tsan_func_entry(pc); ...
}
```

<a name="n1"></a>
**N1, inline hit test.** Most checks find that the same thread already recorded the same access in the shadow cell
and return. N1 does that comparison in place and calls the runtime only on a miss. It costs code: about 25 machine
instructions per check.

```c
x = p->field;
// before:  __tsan_read4(&p->field);
// after:   if (*shadow(&p->field) != expected_record_of_this_thread)
//              __tsan_read4_miss(&p->field);
```

<a name="n1-place"></a>
**Where the hit test is placed (N1-L, K4, s1, s4, k2, S56).** The same test at fewer checks, to save code and compile
time; everywhere else the plain runtime call stays.

| name | the test is inlined only |
|---|---|
| N1-L, K4 | in loops with at most 20 checked accesses |
| s1, s4 | at checks whose block runs at least 1× / 4× per function entry by static frequency |
| k2 | at the 2 statically hottest checks of each function |
| S56 | in functions of at most 56 IR instructions (small accessors) |

<a name="dynstc-rt"></a>
**DynSTC-RT.** DynSTC's idea inside the runtime instead of the compiled code: while one thread is alive the runtime
records nothing. It is a runtime option; the binary is unchanged.

<a name="n1-st"></a>
**N1-ST.** With DynSTC-RT on, the inline test itself is skipped while the program has one thread.

```c
// N1:     if (*shadow(p) != expected) __tsan_read4_miss(p);
// N1-ST:  if (multithreaded && *shadow(p) != expected) __tsan_read4_miss(p);
```

<a name="vwide"></a>
**VWIDE-loops.** [DE](#de)'s verified form extended to checks inside loops that no earlier check dominates: the check
becomes an inline test that a covering record is present, and the full check runs only if it is not.

```c
for (i = 0; i < n; i++)
    sum += a->total;                // before: __tsan_read4(&a->total) per iteration
                                    // after:  inline test per iteration, runtime call only on a miss
```

<a name="memintr"></a>
**MEMINTR, one-sided memory intrinsics.** A `memcpy` whose source is a constant or a private object checks only its
destination.

```c
char buf[64];                       // private to this thread
memcpy(shared->data, buf, 64);      // before: both ranges checked;  after: only shared->data
```

### Refinements of the analyses

<a name="ea-contents"></a>
**EA-CONTENTS.** Reading a pointer out of a private container and publishing that pointer no longer makes the
container itself shared.

```c
item *slots[8];                     // private array
publish(slots[i]);                  // the item escapes; the array does not
slots[j] = 0;                       // before: __tsan_write8(&slots[j]);  after: nothing
```

<a name="swmr-roots"></a>
**SWMR-ROOTS (SWMR thread roots).** SWMR with thread entry functions as roots: reads of a global need no checks if
its only write happens before every thread that can read it is created, even if other threads already exist.

```c
settings.port = 11211;              // the only write
pthread_create(&t, 0, worker, 0);   // worker and what it calls are the only readers
// in worker:  use(settings.port);  // before: __tsan_read4(&settings.port);  after: nothing
```

### Ownership checked at run time

<a name="evconf"></a>
**EVCONF, event-loop confinement.** In an event-driven server each connection object belongs to one worker thread.
Seven annotated fields name that ownership; the owner's accesses run unchecked while a run-time guard holds, and with
the normal checks otherwise. EVCONF-RANGES extends the guard to `memset`/`memcpy` of an owned object, EVCONF-ARGS
carries it into the hash function's key argument, EVCONF-INTERCEPT covers `memchr`, `strlen` and `bcmp` on the owned
read buffer.

```c
void drive_machine(conn *c) {       // c is owned by this worker thread
    // before: every access to *c checked
    // after:
    if (owner_of(c) == self) {      // guard
        c->rbytes += n;             // unchecked
        memset(c->resp, 0, sz);     // unchecked (EVCONF-RANGES)
    } else { /* the same code with the checks */ }
}
```

<a name="evconf-checked"></a>
**EVCONF-CHECKED.** The same elision with no annotation: the analysis proposes the ownership candidates, and run-time
owner checks plus a marker in the shadow make each elision sound. A failed check voids the run instead of losing a
race silently.

<a name="quiet"></a>
**Quiet threads.** Redis's main thread does nearly all the work while the I/O and background threads sleep. The
compiler derives from the whole program where the other threads are parked; while every one of them has been quiet
since a lock release that the main thread acquired, the main thread skips its checks, range checks included.

```c
// main thread, per command:
if (all_other_threads_quiet()) {    // guard
    reply->len += n;                // unchecked
    memcpy(reply->buf, s, n);       // unchecked
} else { /* the same code with the checks */ }
```

<a name="lo-obj-g"></a>
**LO-OBJ-G, objects owned by a lock.** LO per object instead of per variable: a spec says which lock protects which
object, and accesses to the object are unchecked while the thread holds that lock (tested at run time). The spec is
generated from the program's own assertions such as `assert(sqlite3_mutex_held(pBt->mutex))`, or written by hand.

```c
sqlite3_mutex_enter(pBt->mutex);
pBt->nPage++;                       // before: __tsan_write4(&pBt->nPage)
                                    // after:  if (!held(pBt->mutex)) __tsan_write4(&pBt->nPage);
sqlite3_mutex_leave(pBt->mutex);
```

<a name="lo-mark"></a>
**LO-MARK.** LO-OBJ-G without a spec and without assertions: an access made under a lock leaves a marker of that lock
in the shadow, and later accesses under the same lock skip the check while the marker stands. Measured once (1.26× on
SQLite); not adopted.

<a name="cr1"></a>
### Chromium configurations

| name | what is switched on |
|---|---|
| Sound | [STC](#stc), [SWMR](#swmr), [LO](#lo), [EA](#ea) |
| AllOpt | Sound + [DE](#de) + [peeling](#peel) |
| CR1 | Sound + [SWMR-ROOTS](#swmr-roots) + [FE-INL](#fe-inl) + [MEMINTR](#memintr) |
| CR1+N1 | CR1 + [N1](#n1) at every check |
| S56 | CR1 + [N1](#n1-place) only in functions of at most 56 IR instructions |
| library-facts table | a list of facts about library functions for the analyses; CR1 reads the same without it |

<a name="method"></a>
## 7. Method

- Applications: stock and the optimized line are built by the same compiler and run in the same leg, alternating;
  the figure is the geometric mean over 4 code offsets. Each leg carries an A/A arm (the base binary twice).
- Chromium: one build per configuration, no code offsets; arm order rotated between repetitions; intervals are
  bootstrap over stories; TSan reporting off in timed runs.
- Race preservation: 10 stock against 10 optimized runs per app, no race-report site lost (memcached 4 sites kept,
  SQLite 3, MySQL 232-236, Redis and FFmpeg seeded races, since their stock runs report none).
- Compilers: `tsan-cr-8345a0396fa5` (camera-ready root), `tsan-cr2-429bd8cf76b9` (the same plus the static placement
  rules), `tsan-n1s-e52d8787b59e` (the same line with Redis's phase derivation).
- Premises used by results of record: `soundness-fixes.md` §6.
