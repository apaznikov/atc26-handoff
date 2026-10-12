# TSan instrumentation optimizations: results

State: 12 Oct 2026. This is the short form. The full ledger with every leg, control and dead end is the previous
version of this file, commit `9126669`. Soundness, premises and audits: [`soundness-fixes.md`](soundness-fixes.md).

## Contents

| § | section | what it answers |
|---|---|---|
| 1 | [Summary](#summary) | the best figure per application, its configuration and its compile time |
| 2 | [Chromium](#chromium) | every configuration on two hosts and two story samples; what was learned |
| 3 | [Single optimizations](#single) | what each optimization adds alone, per application |
| 4 | [No gain](#nogain) | what was tried and closed, and where the analyses stay conservative |
| 5 | [Compile time](#compile) | build time and speedup of each configuration |
| 6 | [What each optimization does](#explain) | a short text and a before/after C example for every name used here |
| 7 | [Method](#method) | how the figures were taken |

<a name="notation"></a>
**Notation** used in the tables:

| mark | meaning |
|---|---|
| <a name="kind-s"></a>S | static: decided at compile time, nothing added at run time |
| <a name="kind-r"></a>R | checked at run time, no annotation: the compiler inserts a guard or a test |
| <a name="kind-g"></a>G | uses a spec generated from the program's own assertions |
| <a name="kind-a"></a>A | uses a hand-written annotation or spec (A+R, G+R: guarded at run time as well) |
| <a name="host-f"></a>`f` | measured on the Intel host; without the mark, on the AMD host |
| <a name="aa"></a>A/A | the same binary run as two arms of one leg; its ratio shows the noise of the leg |
| **bold** | in section 3: a gain beyond the A/A; elsewhere: the figure of record |
| — | not measured |
| <a name="samples"></a>first / hold-out | Chromium's two story samples: 49 stories used during development / 48 other stories fixed before they ran |

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
| Chromium | blink_perf, [hold-out](#samples) sample of 48 stories; Intel host | 1.39× | **1.10×** ([CR1](#cr1)), **1.16×** ([S56](#n1-place)), **1.25×** ([CR1+N1](#cr1)) | — | — | CR1 +1 %; CR1+N1 +10 % |

- Hosts: AMD = 2 × EPYC 9115 (every row except Chromium); Intel = Xeon w9-3495X (Chromium; its AMD figures are in
  [section 2](#chromium)).
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
sample: 48 other stories, fixed before they ran ([notation](#samples)). Speedup over stock TSan, N=3 unless noted.

| configuration | Intel, first | Intel, hold-out | AMD, first | AMD, hold-out |
|---|---|---|---|---|
| [Sound](#cr1) (the four analyses) | 1.006 (N=2) | — | 0.995 | — |
| [AllOpt](#cr1) (+ dominance elimination, peeling) | 0.994 (N=2) | — | 1.011 | — |
| [CR1](#cr1) | 1.104 | 1.101 | 1.068 | 1.057 |
| [CR1](#cr1) without the [library-facts table](#cr1) | 1.102 (N=2) | 1.106 (N=2) | 1.070 | — |
| [S56](#n1-place) | 1.186 | 1.157 | 1.086 | 1.054 |
| [CR1+N1](#cr1) | 1.286 | 1.248 | 1.081 | 1.017 |

- **The paper's analyses give about 0 on Chromium** on both hosts. The gain comes from the form of the
  instrumentation ([FE-INL](#fe-inl) in CR1 and the inline hit test [N1](#n1)), which is not in the reviewed paper.
- **[CR1](#cr1) holds on unseen stories**: 1.10× on Intel, 1.06× on AMD.
- **The [inline hit test](#n1) pays only on Intel**: +13 to +17 % over CR1. On AMD's hold-out it is 4 % below CR1 (four
  readings of the same sign). [S56](#n1-place) equals CR1 on AMD.
- **Why the hosts differ is not established.** Excluded: thread count (Intel pinned to 16 threads reads the same),
  the container, fonts and paths (AMD on the host reproduces the container within 0.006). Counters show the hit test
  removing cycles in the runtime and adding about as many outside it on both hosts.
- **[Peeling](#peel)** costs 1.8 % on Intel and gains 1.1 % on AMD; each sign is stable on its host.
- **The [library-facts table](#cr1) is not needed**: CR1 without it reads the same everywhere.
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

Change of speed in percent from adding one optimization. Rows down to MEMINTR: over [the paper's analyses](#paper)
with every soundness fix. Rows below: on top of their app's configuration.

Marks S, R, G, A, `f`, bold and "—" are explained under [Notation](#notation).

| optimization | kind | SQLite | memcached | Redis | MySQL | FFmpeg |
|---|---|---|---|---|---|---|
| [DynSTC-RT](#dynstc-rt) | [R](#kind-r) | +0.4 | −0.3 | [`f`](#host-f) −2.2 | +0.1 | [`f`](#host-f) **+12.0** |
| [DynSTC](#dynstc) (the paper's guard) | [R](#kind-r) | — | −0.5 | −5.5 | −0.7 | [`f`](#host-f) **+10.1** |
| [N1](#n1) | [S](#kind-s) | −0.8 | +0.3 | [`f`](#host-f) +2.5 | −1.7 | [`f`](#host-f) **+5.0** |
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
| [LO-OBJ-G](#lo-obj-g), generated spec (over stock) | [G](#kind-g)+[R](#kind-r) | **+15.7** | — | — | — | — |
| [LO-OBJ-G](#lo-obj-g), hand-written spec (over stock) | [A](#kind-a)+[R](#kind-r) | **+25.0** | — | — | — | — |

**Chromium**, step by step (each step over the configuration named in "over"; first sample unless noted; N=2-3):

| step | kind | over | Intel | AMD |
|---|---|---|---|---|
| [The four analyses](#paper) (Sound) | [S](#kind-s) | stock | +0.6 | −0.5 |
| [DE](#de) | [S](#kind-s) | Sound | +0.6 | +0.4 |
| [Peeling](#peel) | [S](#kind-s) | Sound + DE | −1.8 | +1.1 |
| [SWMR-ROOTS](#swmr-roots) + [FE-INL](#fe-inl) + [MEMINTR](#memintr) (= [CR1](#cr1)) | [S](#kind-s) | Sound | **+9.5** | **+7.4** |
| [library-facts table](#cr1) | [S](#kind-s) | CR1 without it | 0.0 | −0.2 |
| [N1](#n1) at every check | [S](#kind-s) | CR1 | **+16.5**; hold-out **+13.4** | +1.2; hold-out −3.8 |
| [N1 in small functions, S56](#n1-place) | [S](#kind-s) | CR1 | **+7.4**; hold-out **+5.1** | +1.7; hold-out −0.3 |
| [N1 in small loops, K4](#n1-place) (N=1) | [S](#kind-s) | CR1 | +0.2 | −0.4 |

<a name="nogain"></a>
## 4. No gain

Tried and closed; one row per family. Shares are of executed checks. Details of each: commit `9126669`, tables 3 and 6.

| family | what was tried | result |
|---|---|---|
| [Dominance elimination](#de), further rules | covered checks deleted instead of [verified](#de); one range check per loop; adjacent fields merged; a cover on every path; covers across loop iterations; post-dominance with proven termination; stronger alias analysis | within ±1 % on every app (Redis −3.8 % for deletion); each rule reaches ≤ 2 % of checks |
| Loop guard (a relative of [peeling](#peel)) | a loop-invariant access checked once per sync-free stretch | Redis +2 to +3 %, others ≈ 0; rejected as imprecise |
| [Escape analysis](#ea), finer | whole-program summaries; per-call-site parameter facts; slots and field chains; fresh return values; thread-local roots | each < 2-3 % of checks; the mass is heap reachable from shared structures, which no static rule reaches |
| [STC](#stc), [SWMR](#swmr), [LO](#lo), finer | closed-world facts; join-aware STC; thread roles; publish-once; written-before-publication; POSIX calls transparent to LO | each < 3 % of checks; two variants lost races and were withdrawn |
| [N1](#n1) variants | compact or batched tests; shared thread-state load; test for relaxed atomics; out-of-line entries saving fewer registers | ±1 %, or −2 to −4 %; the batched form loses races |
| [N1](#n1) placement for speed | profile hot list; static frequency; loops only | none beats the full version in speed; they trade speed for compile time ([section 5](#compile)) |
| [FE-INL](#fe-inl) variants | lazy entry; only hot functions; one exit; no entry/exit in functions without accesses | ±1 %, MySQL −5 % for one exit; the last changes report stacks and was not taken |
| [N1](#n1), [FE-INL](#fe-inl), [MEMINTR](#memintr) on top of SQLite's [LO-OBJ-G](#lo-obj-g) | each added alone | FE-INL +4.5 %, MEMINTR +3.5 % inside a wide [A/A](#aa); N1 −18.5 % |
| More optimization before instrumentation | LTO; Attributor; extra passes | nothing; the Attributor miscompiles |
| Ownership ([EVCONF](#evconf), [LO-OBJ-G](#lo-obj-g), [quiet threads](#quiet)), further | owner tags without a record; memcached's flag globals; MySQL connection fields; SQLite private connections; pager and WAL under the b-tree's mutex; quiet mode on the other four apps | unsound, or needs a premise that was declined, or ≤ 2.4 % of checks |
| The old "same location" rule of [DE](#de) | fields of one struct or elements of one array cover each other | unsound: loses a memcached race stock reports (ceiling +7 to +31 %) |
| Runtime-only changes (as [DynSTC-RT](#dynstc-rt) is) | covering records count as hits; cheaper range checks; cheaper acquires; deadlock-detector cost; allocation bookkeeping | separate track, not in the paper: [`runtime/ideas.md`](runtime/ideas.md) |

Where the analyses stay conservative (share of executed checks they cannot remove):

| what cannot be proven | SQLite | memcached | Redis | MySQL | FFmpeg |
|---|---|---|---|---|---|
| [DE](#de): same address twice, no acquire between, both kept | 3.4 % | 0.04 % | 3.4 % | 5.5 % | 5.9 % |
| [DE](#de): neither check dominates the other | 4.4 % | 0.08 % | 5.8 % | 1.0 % | 8.6 % |
| [EA](#ea): memory that is in fact one-thread or consistently ordered | 72 % | 55 % | 63 % | 49 % | 70 % |
| of it: heap reachable from shared structures | 79 % | 34 % | 42 % | — | 7.5 % |

<a name="compile"></a>
## 5. Compile time

Fresh trees, no ccache, 8 jobs (MySQL 6), stock and the line alternating, medians of 3, wall seconds. For Redis and
memcached the line includes the whole-program summary step.

| app | configuration | speedup | stock build | optimized build | compile-time overhead |
|---|---|---|---|---|---|
| memcached | no annotations (analyses + [EA-CONTENTS](#ea-contents) + [SWMR-ROOTS](#swmr-roots) + [FE-INL](#fe-inl) + [MEMINTR](#memintr)) | 1.02× | 7.25 s | 8.03 s | +11 % |
| memcached | with [EVCONF](#evconf) | 1.81× | 6.47 s | 8.71 s | +35 % |
| Redis | [FE-INL](#fe-inl) + [N1](#n1) + [quiet threads](#quiet) | 1.49× (fresh build; the row of record 1.54×) | 5.37 s | 14.38 s | **+168 %** |
| SQLite | [LO-OBJ-G](#lo-obj-g), generated spec | 1.19× | 33.23 s | 42.57 s | +28 % |
| FFmpeg | [N1](#n1) at every check | 1.285× | 76.10 s | 149.88 s | +97 % |
| FFmpeg | [N1 by static rule s1](#n1-place) | 1.241× | 76.10 s | 110.77 s | +46 % |
| FFmpeg | [N1 by static rule s4](#n1-place) | 1.227× | 76.10 s | 103.53 s | +36 % |
| MySQL | Release, [FE-INL](#fe-inl) | 1.13× | 874.75 s | 1015.13 s | +16 % |
| Chromium | [CR1](#cr1) (400 sampled units, CPU time) | 1.10× ([`f`](#host-f)) | — | — | +1.1 % (1.004-1.018) |
| Chromium | [CR1+N1](#cr1) (same sample) | 1.25× ([`f`](#host-f)) | — | — | +10.4 % (1.076-1.142) |

Redis's build time includes the summary generator with kept inputs and parallel analyses (adopted 12 Oct; the binary
is the same as before, when the overhead was +230 %).

- **The cost is the [inline hit test](#n1)**: about 25 machine instructions and 133-140 bytes per check. Redis by CPU
  time over stock: analyses +1.6 %, + [elimination](#de) and [peeling](#peel) +17 %, + [inlined entry/exit](#fe-inl) +24 %,
  + hit test +157 %.
  The pass itself is 1 % of the added time; the rest is the back end working over the larger code.
- **No change of the pass lowers it**: prototypes of a smaller test gave at most 1-2 % (an assembly form −10 % with
  1.5-1.9 % more code). The one design that would is a back-end pseudo-instruction, recorded as an idea in
  [`runtime/ideas.md`](runtime/ideas.md) and not built.
- **Fewer tests cost speed on Redis** (AMD, over stock): record 1.49×; [static rules](#n1-place) s1 1.34× (+73 % CPU
  time), k2 1.32× (+46 %), s4 1.29× (+42 %); no hit test 1.28× (+26 %).
- **FFmpeg**: no static rule separates the hot checks from the bulk (s1 keeps 48 % of the hot locations for 23 % of
  the code growth; function-size caps keep 2-13 %). The whole speed difference is in the stream-copy transcode.
- **SQLite** by CPU time: record +37 %; without [peeling](#peel) +27 %; without [elimination](#de) and peeling +20 %, at the same
  speed. The owner checks and the analyses are those 20 points.
- **Redis's summary step** (the whole-program pass that [quiet threads](#quiet) need) was 7.9 s of the 17.8 s (+230 %). Not rerunning `make -n` and the dependency build, and
  running the whole-program analyses in parallel, brings it to 4.6 s. Summaries and server code are identical.
- Being measured (12 Oct): where the hit-test hits concentrate on Redis and FFmpeg, to place the test by a profile
  list or per unit.

<a name="explain"></a>
## 6. What each optimization does

The examples are schematic C. `__tsan_read4(p)` / `__tsan_write4(p)` stand for TSan's check of a 4-byte access; real
entry names and the inlined code differ in detail. Each entry: what it does, then the code before and after.

| group | entries |
|---|---|
| [The paper's analyses](#paper) | [STC](#stc) · [SWMR](#swmr) · [LO](#lo) · [EA](#ea) · [DE](#de) · [Peeling](#peel) · [DynSTC](#dynstc) |
| [The form of the instrumentation](#form) | [FE-INL](#fe-inl) · [FE-SINK](#fe-sink) · [N1](#n1) · [N1 placement: N1-L, K4, s1, s4, k2, S56](#n1-place) · [DynSTC-RT](#dynstc-rt) · [N1-ST](#n1-st) · [VWIDE-loops](#vwide) · [MEMINTR](#memintr) |
| [Refinements of the analyses](#refine) | [EA-CONTENTS](#ea-contents) · [SWMR-ROOTS](#swmr-roots) |
| [Ownership checked at run time](#own) | [EVCONF](#evconf) · [EVCONF-CHECKED](#evconf-checked) · [Quiet threads](#quiet) · [LO-OBJ-G](#lo-obj-g) · [LO-MARK](#lo-mark) |
| [Chromium configurations](#cr1) | Sound · AllOpt · CR1 · CR1+N1 · S56 · library-facts table |

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
it and nothing between them can synchronize. In the builds of record the covered check is not deleted but *verified*:
it becomes a short inline test that the earlier record is still in the shadow cell, and calls the runtime only if not.

```c
x = p->len;                         // __tsan_read4(&p->len)
y = p->len + 1;                     // before: __tsan_read4(&p->len);  after: covered by the check above
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
for (i = 1; i < n; i++) s += *p;    // covered
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
and return. N1 does that comparison in place and calls the runtime only on a miss. It costs code: about 25 machine
instructions per check.

```c
x = p->field;
// before:  __tsan_read4(&p->field);
// after:   if (*shadow(&p->field) != expected_record_of_this_thread)
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

DynSTC's idea inside the runtime instead of the compiled code: while one thread is alive the runtime
records nothing. It is a runtime option; the binary is unchanged.

---

<a name="n1-st"></a>
#### N1-ST

With DynSTC-RT on, the inline test itself is skipped while the program has one thread.

```c
// N1:     if (*shadow(p) != expected) __tsan_read4_miss(p);
// N1-ST:  if (multithreaded && *shadow(p) != expected) __tsan_read4_miss(p);
```

---

<a name="vwide"></a>
#### VWIDE-loops

[DE](#de)'s verified form extended to checks inside loops that no earlier check dominates: the check
becomes the inline hit test of [N1](#n1) where an earlier check of the same address in the same epoch is likely (a
static estimate), and only in loop blocks. An unconditional access in a loop needs none of this: peeling and DE cover
it.

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

LO-OBJ-G without assertions or hand annotations; the object's lock is still known per type (where the
object's owner pointer is and which field of the owner is the lock). A guard before the object's accesses tests
whether the object's first shadow cell holds the marker of the lock this thread holds now; if not, the runtime checks
that the thread holds the owner's lock and marks the object, else the accesses stay checked. The marker stays after
unlock. A checked access by a thread that does not hold the lock and meets a marker voids the run, so an unprotected
access is never lost silently. Measured once (1.26× on SQLite); not adopted.

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
| CR1 | Sound + [SWMR-ROOTS](#swmr-roots) + [FE-INL](#fe-inl) + [MEMINTR](#memintr) |
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
  rules), `tsan-n1s-e52d8787b59e` (the same line with Redis's phase derivation).
- Premises used by results of record: `soundness-fixes.md` §6.
