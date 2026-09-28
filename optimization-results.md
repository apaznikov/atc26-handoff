# 🧵 TSan instrumentation optimizations — results

*State as of 28 Sep 2026, 16:20 (focs time). Preliminary: most cells come from one to three sessions at N = 3-4.*

> [!TIP]
> **Headline gains (race-preserving, measured together in one binary)**
> - 🎬 **FFmpeg: +33.5 %** over the base: DynSTC-RT + FE-INL + N1 + N1-ST front (3 sessions). Almost all of it comes from the single-threaded stream copy (×2.96).
> - 🗄️ **Redis: +9.1 %** over the base: FE-INL + N1 (2 sessions).
> - 🐬 **MySQL on AMD: +4.5 %** over the base: FE-INL + VWIDE-loops.
> - ⚙️ Biggest single levers: **N1-ST front +16.9 %** and **DynSTC-RT +12.0 %** on FFmpeg, **FE-INL + VWIDE-loops +6.4 %** on Redis.

> [!WARNING]
> **Where nothing helps yet:** SQLite, memcached, and MySQL on Intel, where FE-INL *loses* 8-9 %. The four levers built for these apps and timed on 27-28 Sep gain nothing: LIBCALL-INLINE, N1-CSE and N1-ATOMIC stay within ±1 %, and FE-SINK loses 1-2 % there (it gains only on Redis and MySQL, table 1).
> The 28 Sep censuses show where memcached's cost actually sits, on the runtime side (the separate runtime track, `runtime/README.md`):
> - 97.6 % of its range-check cells miss, ≈ 18 % of its user CPU;
> - per run, the runtime preempts thread slots ~14 M times and wipes all shadow memory 456 times.

> [!CAUTION]
> **Soundness and correctness**
> - **EA-P5/P6/P7:** three pre-existing holes in today's escape analysis were reproduced (stock 10/10, EA 0/10). The fix is designed and waits for a decision (table 4).
> - **Premise P5, waiting for a ruling.** P5 states that four things are arbitrary: the evicted shadow slot, the recycled trace part, which slot an attach takes, and when a thread re-locks its slot. It covers every lever that removes trace events, the paper's analyses included. The slot counts show it matters: memcached has ~14 M preemptions per run, and each opens a window in which stock TSan already loses races.
> - **MySQL server deaths.** There were 2 deaths (an InnoDB assertion, a lost connection) in 104 runs of N1-family arms (any arm with N1's inline test), and 0 in the other 342 runs (p ≈ 0.06). Four checks of N1 found nothing:
>   - a source review;
>   - an instruction-level diff of 517 InnoDB functions;
>   - an ABI scan of 334 miss calls;
>   - the LLVM machine verifier.
>
>   The most likely cause is an InnoDB race that TSan's timing exposes. It is monitored in every MySQL run.

> [!NOTE]
> **Measurement resolution on MySQL.** On apollo, two builds read 0.989 and 0.954 of the base, although their program code is byte-identical. The two builds come from different compiler roots.
> - The layout control rules out the program's layout on apollo. The base with its data shifted by a page reads 1.003, and with its code shifted by 480 B or by 512 B it reads 1.005 (4 rotated runs each).
> - The gap therefore comes from the root, most likely its TSan runtime. A test that swaps the runtimes between the two builds is queued, and the same layout control on Intel (focs) runs tonight.
> - Until they report, read MySQL differences between roots below ~3.5 % with care. Within one root, layout moved MySQL by ≤ 0.5 % on apollo.
> - The debug build embeds the tree path through `__FILE__`, so every arm now gets a tree path of equal length.

---

## 🧭 How to read the tables

| marker | meaning |
|---|---|
| 🟢 **+x** | clear gain, ≥ +2 % |
| 🟡 ±x | small effect (1 … 2 %), or unresolved `?` |
| ⚪ x | noise, within ±1 % |
| 🔴 −x | loss, ≤ −2 % |
| `f` / `a` | measured on **focs** (Intel Xeon w9-3495X, 112 threads) / **apollo** (2 × AMD EPYC 9115) |
| `?` | sessions disagree (unresolved) |
| screening | measured next to a busy neighbour; a lead, not a reading of record |
| — | not measured |

- A number is the speed gain in percent, (speed ratio − 1) × 100. Resolution is about 2-4 % (MySQL about 5 %, see the note above).
- **The reference.** Unless a row says otherwise, the reference is the base **P1-v3**.
  - P1-v3 is the shipped configuration (EA, LO, STC, SWMR, and DE with loop peeling), with DE made **exact** plus every soundness fix found after shipping.
  - Exact means "verified removal": a check is removed only when an inline copy of TSan's hit test proves that the check would hit.
  - P1-v3 over the shipped artifact: SQLite +0.3, memcached +1.4, Redis −2.9, MySQL 0…+3, FFmpeg −0.3.
- New levers are measured over a control built from the same compiler root with the lever switched off.
- ✅ **Race-preserving.** Everything in tables 1-3 loses no race that stock TSan reports, under the project's stated premises. Each was checked with IR tests, check-tsan, reproducers with controls, and an independent audit.
- 📖 Each table's key explains every name used in that table. General terms:
  - **TSan terms.** A **check** is the call inserted before a memory access (`__tsan_read4`, …). A **shadow cell** records recent accesses to an 8-byte granule: a **hit** means the access is already recorded, a **miss** means the runtime must check for races and record it. The **trace** is the per-thread log used to restore stacks in race reports. **Epoch / slot / preemption / DoReset** are TSan v3's per-slot logical clocks, the 256 slots shared by all threads, the taking of a live thread's slot, and the global reset that wipes all shadow. An **interceptor** wraps a libc call; a **range check** covers a byte range.
  - **The paper's analyses.** **STC**: no checks in code that can only run before threads exist. **SWMR**: if every write to a variable happens in single-threaded context, its reads need no check. **LO**: if every multi-threaded access to a variable holds a common lock, those accesses need no check. **EA**: no checks on memory that never becomes reachable from another thread. **DE**: a check dominated by an identical one with no synchronisation in between is dropped; **loop peeling** copies a loop's first iteration so the body is dominated. **DynSTC**: a run-time thread counter that skips checks while the process is single-threaded. **AllOpt**: all of them together. **stock**: unmodified TSan. **shipped artifact**: the compiler submitted with the paper. **tier A (T0-T11)**: the 25 Sep sweep over the shipped artifact (T0 stock, T1 shipped, T10 all new levers including inexact DE).
  - **Measurement.** Each machine has two benchmark halves; a timed **leg** owns one. **idle**: the other half is empty. A **session** is one sitting, an **arm** one binary, a **cell** one run of one arm, **N** the runs per arm, and **n = k** means only k runs survived. **ABBA / rotated**: arm order alternates so drift hits all arms alike. **Control / same-root control**: the arm from the same compiler with the lever off. **CV clause tripped**: a subtest's run-to-run spread exceeded its limit, so the leg is re-run. **pend. / re-run queued**: not yet measured, or being repeated. **= base / = FE**: the flag changes no code for that app. **M/s, G**: millions per second, billions.
  - **Workloads.** SQLite: threadtest3, where **stable-4** is the geomean of walthread1, walthread2, checkpoint_starvation_1 (**cs1**) and stress2 (bimodal stress1 is left out); **all-7** is the geomean of all seven subtests. memcached: memtier_benchmark. Redis: redis-benchmark. MySQL: sysbench OLTP (Select = oltp_read_only, Write-only = oltp_write_only, read-write, point and range selects). FFmpeg: transcodes of one film (copy = single-threaded stream copy; h264, h265, **mjpeg** = encoders).
  - **Premises and audits.** **P-EV**: which record TSan's bounded shadow evicts is arbitrary. **P5**: P-EV widened to the recycled trace part, the slot an attach takes and the moment of a re-lock (waiting for a ruling). **A3**: signal handlers do not synchronise. **A3-fiber**: a handler returns on the same fiber (not adopted). **Axx** (A21, A23, …): numbered independent audits; "sound with conditions" means correct if the listed conditions hold. **idea 6-9**: item numbers in Alexey's list of 27 Sep.

---

## 🟢 Table 1 — optimizations that gain somewhere

| optimization | SQLite | memcached | Redis | MySQL | FFmpeg |
|---|---|---|---|---|---|
| **FE-INL** | `a` ⚪ −0.7 | `a` ⚪ +0.8 | `f` 🟢 **+4.8** | `f` 🔴 −8.9 · `a` 🟢 **+4.2** | `f` ⚪ 0.0 |
| **FE + VWIDE** | `a` ⚪ 0.0 | `a` ⚪ −0.3 | `f` 🟢 **+5.2** | `f` 🔴 −8.6 · `a` 🟢 **+4.5** | `f` 🟢 **+2.0** |
| **FE + VWIDE-loops** | `a` ⚪ +0.1 | `a` ⚪ +0.6 | `f` 🟢 **+6.4** | `f` 🔴 −7.6 · `a` 🟢 **+4.5** | `f` 🟡 +1.8 |
| **N1** | `f` 🔴 −2.1 `?` · `a` ⚪ −0.8 | `f` 🔴 −2.7 · `a` ⚪ +0.3 | `f` 🟢 **+3.7** | `f` 🟡 −1.4 · `a` 🟡 −1.7 | `f` 🟢 **+5.0** |
| **N1-L** | `f` ⚪ 0.0 `?` · `a` ⚪ +0.3 | `f` 🟡 `?` · `a` ⚪ −0.9 | `f` 🔴 −2.5 | `a` ⚪ −0.1 | `f` 🟢 **+3.0** |
| **DynSTC-RT** | `f` 🟡 +0.8 `?` · `a` ⚪ +0.4 | `f` 🟡 +2.6 `?` · `a` ⚪ −0.3 | `f` 🔴 −2.2 | `a` ⚪ +0.1 | `f` 🟢 **+12.0** |
| **N1-ST front** ¹ | `f` ⚪ 0.0 `?` · `a` ⚪ +0.2 | `f` ⚪ −0.8 · `a` ⚪ −0.2 | `f` 🟡 −1.4 | `f` 🟡 −1.5 · `a` 🟡 −1.8 | `f` 🟢 **+16.9** |
| **N1-ST miss** ¹ | `f` 🔴 −4.3 `?` · `a` 🟢 **+2.0** | `f` 🔴 −2.8 · `a` ⚪ +0.1 | `f` 🟢 **+2.1** | `f` ⚪ −0.4 · `a` 🔴 −2.6 | `f` 🟢 **+9.8** |
| **MEMINTR** | `a` 🟡 +0.7…+2.5 (2 sessions; cs1 +3.8 in both) · `f` screening | `a` ⚪ +0.9 · `f` ⚪ −0.4 (screening) | `f` 🔴 −2.5 (idle; perf: no cycle change, likely noise or layout) | `f` 🟡 +1.6 (screening) · `a` re-run queued | `f` ⚪ 0.0 |
| **FE-SINK v2** ³ | `a` ⚪ +1.1 (stress2 alone); with FE-INL 🟡 −1.0 | `a` 🟡 −1.7; with FE-INL 🔴 −2.2 | `f` 🟢 **+4.1** (call entries; FE-INL alone +6.4); with FE-INL 🟡 −1.3 | `a` 🟢 **+2.8** (call entries; FE-INL alone +5.6); with FE-INL ⚪ +0.4 | `f` 🟡 −1.2; with FE-INL 🟡 −1.0 |
| **exact DE package** ² | `a` ⚪ +0.8 | `f` 🟢 **+3.6** | `f` 🔴 −2.6 | `f` 🟢 **+3…+6** | `f` ⚪ +0.5 |
| **LO-OBJ-G** ⁴ | `a` 🟢 **+18.2** (stress2 alone); stable-4 🟡 +1.2, all-7 🟡 +1.3; walthread1 🔴 −2.9 | ≤ +0.2 (census) | — | ≈ 0 (census) | — |

<sub>¹ Measured on top of N1 with DynSTC-RT; on FFmpeg, on top of DynSTC-RT + FE + N1; on Redis, without DynSTC-RT. ² Measured over the shipped artifact (tier A), not over P1-v3. ³ Over its same-root control (P1-v3 + N1); all readings of record. ⁴ Over P1-v3 in the same leg (apollo half B, 32 CPUs, ABBA N = 6). Relies on premises R3 and P-OWN and on annotations; not yet adopted. stress2 is from the root with the A24b fixes, the composites from the root before them (the fixes add no work on a hit).</sub>

**🔎 Key to table 1**
- **FE-INL** (inline function entry/exit; **FE** in combinations). The push and pop of TSan's shadow call stack are inlined instead of calling `__tsan_func_entry/exit`; the trace events are identical.
  - It gains on Redis and on MySQL on AMD.
  - On MySQL on Intel it loses. FE executes up to 8 % fewer instructions per transaction there but no fewer cycles, because the +59 % code size raises i-cache and iTLB stalls. The debug-build MySQL is lock-bound, so longer transactions hold locks longer.
  - *The processor decides the sign, contention the size.* Part of it may be layout (see the note at the top).
- **VWIDE** (idea 7, "cheap run-time DE checks"). Exact removal also at sites that no covering check dominates but that usually hit in practice; an inline hit test verifies the cover at run time. **VWIDE-loops** does this only inside loops. Alone they gain nothing (table 2); here they appear combined with FE.
- **N1**. TSan's fast-path hit test ("is this access already recorded in shadow?") is inlined at every access, and the runtime is called only on a miss. **N1-L**: the same, only in loops with at most 20 checks.
- **DynSTC-RT**. A run-time single-thread mode: while the process has one live thread, the runtime records nothing, and the create/join edges order everything else. Its whole gain is FFmpeg's single-threaded stream copy.
- **N1-ST**. N1's inline test is skipped while the thread is in that single-thread mode, where it would always miss.
  - **front**: the flag is read before the test. This gives the largest gain on single-threaded code and a 1-3 % tax on multi-threaded code.
  - **miss**: the flag is read only on a miss. There is no tax on hits and about half the gain.
- **MEMINTR** (idea 8b). A memcpy/memmove whose source is a constant or an uncaptured local has only its destination checked. Small gains on SQLite (checkpoint_starvation_1 +3.8 % in both apollo sessions) and memcached. Redis read −2.5 %, but a perf check found no mechanism: cycles unchanged on the losing commands (PING_MBULK 1.001, ZPOPMIN 0.995), so it is likely noise or layout.
- **exact DE package**. DE's merging of adjacent checks and its loop ranges, made exact under verified removal.
- **LO-OBJ-G** ⁴ (lock ownership relative to an object, with a run-time guard). Fields of objects owned by one SQLite BtShared (pages, Pager, WAL) are left unchecked while the thread holds that BtShared's mutex. An inline test checks this at run time: the thread's "any mutex held" flag plus a cached last owner, with the runtime called only on a miss. The owners come from annotations taken from SQLite's own `sqlite3_mutex_held` assertions.
  - Premises: **R3**, every access to an annotated object that conflicts with another thread's access holds the annotated lock, or happens before the object is published; **P-OWN**, the annotations name the right lock, and an owner's lock does not change while the owner lives. Audits A24 and A24b: sound under both, with listed conditions.
  - The gain sits in the shared-cache subtests: stress2 +18.2 %, where 97.6 % of the guarded checks are skipped. Subtests with a private BtShared skip nothing and pay ~3 % for the test (walthread1 −2.9 %), so the composites move ~1 %. dynamic_triggers is unresolved (0.74-1.30 per pass).
  - The memcached and MySQL figures are censuses, not timings.
- **FE-SINK v2** ³. The function-entry call is sunk to the first point that needs the frame. It gains only with call entries and only on the two call-heavy servers, Redis (+4.1 %) and MySQL on AMD (+2.8 %), where FE-INL alone gives more (+6.4 %, +5.6 %). On top of FE-INL it adds nothing (MySQL +0.4 %) or costs 1-2 %, so it adds nothing to the best configurations. Audit A23: sound with conditions; premise P5 pending.
  - Arms in its legs: FB is the control, FI is FE-INL alone, FS is sink with call entries, FIS is sink with FE-INL, FIC is FE-INL-CSE (table 2).
  - Counting runs: sinking removes ~22 % of executed entries on FFmpeg and Redis, which is < 1 % of cycles against 2-12 % more code.

---

## ⚪ Table 2 — optimizations that gain nothing (or lose)

| optimization | SQLite | memcached | Redis | MySQL | FFmpeg | verdict |
|---|---|---|---|---|---|---|
| **LIBCALL-INLINE** (light compare entries) | `a` ⚪ 0.0 | `a` ⚪ +0.4 | `f` ⚪ −0.7 (screening) | `a` ⚪ −0.4 | `f` ⚪ +0.3 | **closed** 28 Sep: correct and exact, no effect |
| **N1-CSE** | `a` ⚪ +0.3 | `a` ⚪ −0.4 · `f` ⚪ −1.0 (screening) | — | `a` ⚪ +0.9 · `f` 🟡 −1.6 (screening) | — | no effect on the three apps it was built for |
| **N1-ATOMIC** | `a` ⚪ −0.4 (without stress2 +0.9; CV tripped, re-run queued) | — | — | — | — | no effect on its target (SQLite's relaxed atomics) |
| **FE-INL-CSE** (evidence only) | — | `a` ⚪ +0.4 | `f` ⚪ +0.8 | `a` ⚪ −0.4 | — | no effect; measured over FE-INL; relies on the unadopted A3-fiber premise |
| VWIDE (alone) | `a` ⚪ −0.6 | `a` ⚪ +0.6 | `f` ⚪ −0.5 | `f` ⚪ −0.3 · `a` ⚪ −0.3 | `f` ⚪ +0.8 | noise; no effect together with N1 |
| VWIDE-loops (alone) | `a` ⚪ +0.2 | `a` ⚪ 0.0 | `f` ⚪ −0.9 | `f` 🟡 −1.8 | `f` 🟡 +1.3 | noise |
| N1-PM (N1 + preserve_most) | `a` 🔴 −2.8 · `f` 🔴 −2.1 (n=1) | `f` ⚪ −1.0 · `a` ⚪ −0.2 | `f` 🟢 +3.8 (idle; N1 alone +3.7) | `f` 🔴 −2.2 (screening) · `a` 🔴 −2.5 | `f` 🟢 +5.2 (mjpeg +14) | its gains on FFmpeg and Redis are N1's own (+5.0, +3.7); loses on SQLite/MySQL |
| N1-LOOPS-∞ (N1-L without cap) | `a` 🟡 −1.3 · `f` ⚪ −0.4 (n=2) | `f` 🟡 +1.4 (idle; N1 alone also +1.4) · `a` ⚪ −0.8 (2 sessions) | `f` ⚪ +0.4 (idle) | `f` 🟡 −1.8 (screening) · `a` ⚪ −0.1 | `f` 🟢 +3.3 (mjpeg +9) | the FFmpeg gain is N1-L's own (+3.0) |
| DE-5 (cycle cut) | `a` ⚪ +0.3 | `a` ⚪ −0.1 · `f` ⚪ −0.8 | pend. | `a` ⚪ +0.6 | `f` ⚪ +0.1 | no effect; Redis pending |
| DE-6 (SFI judges calls) | `a` 🟡 +1.3 (stress2 +6) | = base (no code change) | pend. | `a` ⚪ +0.4 | `f` ⚪ 0.0 | no effect; Redis pending |
| DE-7 (directional SFI) | = base | = base | = base | `a` ⚪ +0.4 | `f` ⚪ +0.7 | no effect |
| DE-8 (fence no-sync) | = base | = base | = base | `a` ⚪ +0.5 | `f` ⚪ −0.1 | no effect |
| DE-5..8 together | `a` ⚪ +0.3 | `a` ⚪ −0.1 (= DE-5) | pend. | `a` ⚪ +0.6 | `f` ⚪ +0.4 | no effect; Redis pending |
| FE-INL exit-max=1 | `a` = FE | — | — | `f` 🔴 −5.2 | — | variant, dropped |
| FE-INL unified exit | = FE | = FE | = FE | = FE | = FE | byte-identical to FE on -O2 code, dropped |
| N1b | `f` 🔴 −4.3 · `a` ⚪ +0.8 | `f` 🟡 +3.0 `?` · `a` 🔴 −2.7 | `f` 🟡 −1.3 | `a` 🔴 −3.7 | `f` ⚪ −0.8 | dropped |
| N1-S | `a` ⚪ +0.4 | `a` ⚪ +0.1 | `f` 🔴 −2.2 | `f` 🔴 −2.0 | `f` ⚪ +0.5 | closed |
| WP | `a` 🟡 −1.5 | `a` ⚪ +0.1 | `f` ⚪ +0.6 | ⚪ ≈ 0 (static) | `f` ⚪ −0.3 | no gain |
| N2 ² | `a` 🟡 −1.9 | `f` ⚪ −0.4 | `f` 🟡 −1.7 | dropped | `f` ⚪ −0.1 | reference only |
| SUBS | `a` 🔴 −18.2 | `a` 🔴 −10.9 | `f` 🔴 −2.0 | `a` 🔴 −8.0 | `f` 🔴 −14.7 | ❌ closed |
| loop guard ² ⚠️ not race-preserving | +1.0 | −0.7 | +2.1 | `f` −5.2 | — | 🚫 excluded |
| all inexact T10 ² ⚠️ not race-preserving | −1.2 | +1.7 | +0.9 | `f` −6.6 | +20.5 | 🚫 excluded |

<sub>² Measured over the shipped artifact (tier A), not over P1-v3.</sub>

**🔎 Key to table 2**
- **LIBCALL-INLINE**. Calls from instrumented code to strcmp, strncmp, strcasecmp, memcmp, bcmp, memchr and strlen go to light runtime entries. These record the interceptor's read ranges without its frame.
  - It saves ~3-6 ns per call (microbenchmark, 32 B) against 7-13 ns for the two range checks that both paths pay.
  - At the observed call rates (up to 2.3 M/s) that is ≤ 0.8 % anywhere.
  - Audits A21/A21b: sound with conditions (no LTO; suppression patterns).
- **N1-CSE**. A compact inline hit test: a run of accesses shares the sibling shadow address and the byte mask. It is read over a control built from the same root.
- **N1-ATOMIC**. N1's inline hit test extended to relaxed atomic loads and stores. SQLite's walFindFrame does 151 M of them per run, and 55.5 % of the loads hit.
- **FE-INL-CSE**: one thread-state load per function for FE-INL's entry, exits and inline tests (an audit blocker, the state read before its initialisation, was fixed before timing). Timed as an extra arm in the FE-SINK legs.
- **VWIDE / VWIDE-loops (alone)**: see table 1; here without FE.
- **N1-PM**: N1 whose miss call goes through `preserve_most` entries with a hand-written hit path, so fewer registers are saved at the call site.
- **N1-LOOPS-∞**: N1-L without its limit of 20 checks per loop.
- **SFI** (SyncFreeInfo): DE's per-function summary of whether a call can synchronise, which decides whether a dominating check still covers across the call. **DE-5** (cycle cut): DE's cycle scan only over cycles that avoid the cover. **DE-6**: SFI and the loop-free test judge calls to declarations the way the scan does. **DE-7**: callees that only acquire do not block dominance. **DE-8**: fences count as non-synchronising.
- **FE-INL exit-max=1**: FE-INL with at most one inlined exit per function, a variant for C++ unwinding. **FE-INL unified exit**: returns merged before the exit is inlined; byte-identical to FE on -O2 code.
- **N1b**: no inline test; the runtime call itself goes through cheaper `preserve_most` entry points, which save fewer registers.
- **N1-S** (idea 6): N1 only at statically hot sites, with a budget of 2 per function and no profile. It misses the hot sites of large functions.
- **WP**: whole-program summaries for the static analyses (closed world, `-tsan-external-symbols`).
- **N2**: several checks batched into one runtime call.
- **SUBS**: a same-thread, same-epoch shadow record that covers the access's bytes with an equal or stronger kind counts as a hit, with an absorbing store and exact merging. The extra run-time test on every miss costs more than it saves.
- **loop guard / T10**: inexact DE variants, which remove checks by coverage without run-time verification. They can lose races under TSan's bounded shadow, so they are excluded. The FFmpeg +20.5 % is shown only to mark what soundness costs.

---

## 📊 Table 3 — cumulative results: the best combination per app vs one universal combination

| configuration | SQLite | memcached | Redis | MySQL | FFmpeg |
|---|---|---|---|---|---|
| 🏆 **best per app**, measured together (over P1-v3) | ⚪ nothing gains | `a` ⚪ +0.6 (FE + VWIDE-loops) · `f` nothing | `f` 🟢 **+9.1** (FE + N1; 2 sessions) | `a` 🟢 **+4.5** (FE + VWIDE-loops) · `f` nothing | `f` 🟢 **+33.5** (DynSTC-RT + FE + N1 + N1-ST front; 3 sessions) · +14.2 without N1-ST |
| 🌐 **universal U1** = N1 + N1-ST miss + DynSTC-RT (over P1-v3) | `f` 🔴 −4.6 `?` · `a` 🟡 −1.6 | `f` 🔴 −5.0 | `f` 🟢 **+4.7** | `f` 🔴 −3.5 · `a` 🔴 −4.3 | `f` 🟢 **+25.1** |
| 🌐 **universal U2** = U1 + FE (over P1-v3) | `f` 🔴 −4.4 `?` · `a` 🟡 −1.1 | `f` 🔴 −4.7 | `f` 🟢 **+7.5** | `f` 🔴 −9.1 · `a` 🟡 +1.9 | `f` 🟢 **+25.3** |
| 📄 **the submitted paper**: TSan+AllOpt over *stock TSan* | +71 | +7 | +45 | +16 (Select) · +11 (Write-only) | +57 |
| 🔭 ceiling O1-all / O2-eraser (oracles, not optimizations) | +15 / +52 | +4 / +9 | +18 / +28 | −1 / +3 | +31 / +41 |

**🔎 Key to table 3**
- 🏆 **Best per app**: the best race-preserving combination for that app, with all its optimizations in one binary, measured in one session. It is not a product of single gains.
- 🌐 **Universal U1/U2**: one configuration applied to every app. Against the best per app it gives up about 1.6 points on Redis and about 8 on FFmpeg. It costs SQLite and memcached on focs, where N1's inlined code is slower.
- 📄 **Paper row**: the speedups printed in the submitted paper (TSan+AllOpt over stock TSan; Chromium, not in this table, is 1.39× geomean).
  - They come from earlier compilers, including DE by coverage and the post-dominance heuristic.
  - They use a different reference: *stock* TSan, not P1-v3.
  - For comparison, our best combinations over stock TSan reach about +1 % (SQLite), +4 % (memcached), +4…+11 % (Redis), +8…+13 % (MySQL on AMD) and +33 % (FFmpeg).
- 🔭 **Ceilings**: profile oracles. O1-all skips every check on memory touched by one thread; O2-eraser also skips memory Eraser would exempt by a common lock. They are unsound upper bounds on what an analysis of that kind could reach, not optimizations.
- ⏳ **Batch 11, tonight**: the **optimistic** configuration and the **realistic** one.
  - Where and when: on focs for all apps from ~19:30, and on apollo for SQLite, memcached and MySQL from ~19:05.
  - Optimistic: the best arm per subtest, chosen from earlier sessions and read in a fresh session. The choice is pre-registered in `batch11-preregistration.md`.
  - Realistic: the single arm with the best geomean over all apps.
  - Arms: base, U3 = FE + VWIDE-loops, U4 = FE + N1, U2, and U5 = FE + N1 + N1-ST front.

---

## 🧪 Table 4 — ideas not yet timed, or closed without timing

| idea | what it is | status | ceiling or expected gain |
|---|---|---|---|
| 🛑 **EA-P5/P6/P7** (soundness) | today's EA loses races when a pointer to a local is published through a pipe (write/read), through `%p` text, or through the generic `__atomic_load` libcall | ✔️ reproduced (stock 10/10, EA 0/10 each); fix designed, paused pending a decision | correctness, not speed |
| 🧩 **runtime track** (separate) | ideas that change only TSan's runtime: RT-SYNC, RT-RANGE (RANGE-HIT-SKIP/VEC closed, RANGE-UNIFORM open), RT-ALLOC, RT-SLOT (slot chains and the epoch budget) | kept apart from the paper's track (Alexey 25 and 28 Sep); censuses only, no implementation; overview in `runtime/README.md` | memcached: missed range cells ≈ 18 % of user CPU; 14 M slot preemptions and 456 shadow wipes per run |
| **EA-SLOT** (idea 8a) | a value loaded from a stack slot and passed on escapes as the slot's contents, not the slot itself (memcached's `tokens` array) | ❌ closed 28 Sep by census | at the most optimistic, 0.16 % of memcached's checks (139 sites; not the ≈ 3 % first estimated), 0.004 % of SQLite's, one site in MySQL |
| **CLONE-ESC** (idea 9) | clone functions with hot pointer arguments into escaping and non-escaping versions; choose the clone at run time | timed oracle OA0 (unsound upper bound): SQLite ≤ +2.6 % (screening), memcached ≤ +3.5 %, Redis ≤ +6.0 %, FFmpeg ≤ +10.4 % (the stream-copy part overlaps DynSTC-RT's gain). Sound part by publication profile: memcached 0.04 % of checks (≈ 0.02 % of time), gate shut; FFmpeg dropped (OA0 ≤ 1.8 % on its multi-threaded codecs); Redis pending | ≤ 7 % FFmpeg/Redis, ≤ 3 % SQLite, ≤ 1 % memcached, 0 MySQL |
| 🆕 **MEMINTR-INLINE** | small constant-size memcpy/memset/memmove kept as intrinsics and checked by range entries or inline tests instead of the interceptor call | census + design | limited by the range checks it keeps (see the runtime track) |
| 🆕 **N1-SPLIT** | N1/VWIDE miss blocks marked cold and split to .text.unlikely | compile-only check first | front-end loss on MySQL-f, SQLite, memcached |
| 🆕 **N1-PAIR** | one 32-byte load + movmsk tests two adjacent shadow cells | census | SQLite page/record parsing |
| 🆕 **SPIN-ACQ** | in a loop of only atomic loads + arithmetic, poll relaxed and acquire only when the value changes (Redis getIOPendingCount: 19.1 G seq_cst acquires) | ceiling only (Alexey); can only add reports (ABA), never lose one | Redis ≤ +14 % (io-threads bound) |
| 🆕 **compile time** | compile-time overhead of P1-v3 AllOpt over stock | measured (quiet, one build at a time): CPU user+sys SQLite +20.1 %, memcached +8.4 %, Redis +9.4 %, FFmpeg +15.6 %; worst unit sql_yacc.cc +55 %; N1 inline ×1.8-3.3; MySQL whole still open | paper says 1-18 % |
| N1-ST (b), (c) | hoist the single-mode flag read per call-free run; or encode single mode in the fast state TSan already loads | not built | would halve or remove N1-ST front's 1-3 % multi-threaded tax |
| sound loop ranges | one range check per loop, placed after the loop or per chunk | ❌ closed 27 Sep: no form is sound under P-EV | FFmpeg −12.7 % executed calls; memcached −0.6 %; SQLite ≈ −1 % |
| SWMR-H | a location written only before its publication is exempt afterwards | ⏸️ parked (audit: 14 lost-race paths; needs a premise on indirect calls) | U1 ceiling: memcached +0.2, Redis 0, SQLite 0 |
| LO-F | lock ownership per field | ⏸️ parked with data (ceiling ≈ +0.2 % memcached; blocked like SWMR-H) | ≈ 0.5 % of memcached's checks |
| LO-W | recognise mutexes LO cannot identify today | ❌ closed by ceiling | Eraser ceiling ≤ 1.4 % |
| EA-7 | refine "arguments of address-taken/external functions escape" | open, static count only | +5.5 k static sites |
| EA-WP | EA with whole-program summaries | ❌ census: nothing on 3 apps | ≈ 0 |
| EA-TL / EA-HEAP | thread-local memory / "own heap", statically | ❌ no-go (the objects are reachable from globals) | — |
| SWMR-G | write-once-then-read-only memory | ❌ no static route | shared read-only memory 4-14 % of checks |
| DE-2 | merge adjacent same-kind fields into one ≤ 8-byte check | ❌ closed 27 Sep (Alexey) | ≤ 1.1 % SQLite |
| DE-1, DE-AA, DE-4, DE-9 | more same-address proofs, stronger alias analysis, cover at an offset, constant-data judgments | ❌ census | 0-0.2 % |
| DE-10 / DE-AV | availability instead of strict dominance / a dynamic loop guard for any repeated address | DE-10 covered by VWIDE; DE-AV idea only | — |
| N6 | subsumption-aware shadow eviction | soundness note, open | — |
| STC-1..4, DYN-1 | more single-threaded-context rules; drop DynSTC guards where more threads certainly exist | ❌ oracle bound: nothing hot | ≈ 0 |
| LO-B1/B2 | private mutexes immune to unknown unlocks; callback releases | ❌ unsound, closed | — |

**🔎 Key to table 4**
- **Ceiling**: an upper bound from a profile oracle or a census of executed checks, not a timed optimization.
- **Census**: a static or profile count of the checks or runtime events a rule would remove, weighted by how often they run.
- ⏸️ **Parked**: set aside with its data kept. ❌ **Closed**: measured or argued to have no worthwhile gain, or unsound.
- **Audit**: an independent read-only review against the zero-lost-races rule, done before anything is timed.
- ⭐ marks the one direction with large headroom on an app where nothing else helps (SQLite).
- **Name decoding.**
  - EA-TL / EA-HEAP: thread-local and "own heap" memory. SWMR-H: written only before publication (hand-off). SWMR-G: write-once-then-read-only.
  - LO-OBJ / LO-F / LO-W: lock ownership per owning object, per field, and for wrapped mutexes. LO-B1/B2: see the row.
  - DE-AA: alias analysis. DE-AV: availability guard. DYN-1: DynSTC guards.
  - N6: the sixth item of the N-series (N1, N2, …).
  - C11-C13: fix items from the EA-SLOT audit.
  - BtShared, WAL, Pager: SQLite's shared B-tree state, write-ahead log and page manager.
  - OA0: CLONE-ESC's unsound oracle, which treats every hot pointer argument as non-escaping.
  - ABA: a value changes and changes back unseen.
  - RT-*, RANGE-*: runtime-only ideas; see `runtime/README.md`.
</content>
</invoke>

