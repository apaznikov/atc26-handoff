# 🧵 TSan instrumentation optimizations — results

*State as of 28 Sep 2026, 12:30 (focs time). Preliminary: most cells come from one to three sessions at N = 3-4.*

> [!TIP]
> **Headline gains (race-preserving, measured together in one binary)**
> - 🎬 **FFmpeg: +33.5 %** over the base: DynSTC-RT + FE-INL + N1 + N1-ST front (3 sessions). Almost all of it comes from the single-threaded stream copy (×2.96).
> - 🗄️ **Redis: +9.1 %** over the base: FE-INL + N1 (2 sessions).
> - 🐬 **MySQL on AMD: +4.5 %** over the base: FE-INL + VWIDE-loops.
> - ⚙️ Biggest single levers: **N1-ST front +16.9 %** and **DynSTC-RT +12.0 %** on FFmpeg, **FE-INL + VWIDE-loops +6.4 %** on Redis.

> [!WARNING]
> **Where nothing helps yet:** SQLite, memcached, and MySQL on Intel, where FE-INL *loses* 8-9 %. The four levers built for these apps and timed on 27-28 Sep gain nothing: LIBCALL-INLINE, N1-CSE, N1-ATOMIC and FE-SINK all stay within ±1.6 % (table 2).
> The 28 Sep censuses show where memcached's cost actually sits, on the runtime side (table 4):
> - 97.6 % of its range-check cells miss, ≈ 18 % of its user CPU;
> - per run, the runtime preempts thread slots ~14 M times and wipes all shadow memory 456 times.

> [!CAUTION]
> **Soundness and correctness**
> - **EA-P5/P6/P7:** three pre-existing holes in today's escape analysis were reproduced (stock 10/10, EA 0/10). The fix is designed and waits for a decision (table 4).
> - **Premise P5, waiting for a ruling.** P5 states that four things are arbitrary: the evicted shadow slot, the recycled trace part, which slot an attach takes, and when a thread re-locks its slot. It covers every lever that removes trace events, the paper's analyses included. The slot counts show it matters: memcached has ~14 M preemptions per run, and each opens a window in which stock TSan already loses races.
> - **MySQL server deaths.** There were 2 deaths (an InnoDB assertion, a lost connection) in 104 runs of N1-family arms, and 0 in the other 342 runs (p ≈ 0.06). Four checks of N1 found nothing:
>   - a source review;
>   - an instruction-level diff of 517 InnoDB functions;
>   - an ABI scan of 334 miss calls;
>   - the LLVM machine verifier.
>
>   The most likely cause is an InnoDB race that TSan's timing exposes. It is monitored in every MySQL run.

> [!NOTE]
> **Measurement resolution on MySQL.** On apollo, two builds read 0.983 and 0.954 of the base, although their code is byte-identical. Only the layout differs: the code is shifted +480 B and the data +1 page.
> - A layout control is running now: the data shifted by a page, and the code shifted by 480 B and by 512 B.
> - Until it reports, read MySQL differences below ~3 % with care.
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
| **MEMINTR** | `a` 🟡 +0.7…+2.5 (2 sessions; cs1 +3.8 in both) · `f` screening | `a` ⚪ +0.9 · `f` ⚪ −0.4 (screening) | `f` 🔴 −2.5 (idle; under review) | `f` 🟡 +1.6 (screening) · `a` re-run queued | `f` ⚪ 0.0 |
| **exact DE package** ² | `a` ⚪ +0.8 | `f` 🟢 **+3.6** | `f` 🔴 −2.6 | `f` 🟢 **+3…+6** | `f` ⚪ +0.5 |

<sub>¹ Measured on top of N1 with DynSTC-RT; on FFmpeg, on top of DynSTC-RT + FE + N1; on Redis, without DynSTC-RT. ² Measured over the shipped artifact (tier A), not over P1-v3.</sub>

**🔎 Key to table 1**
- **FE-INL** (inline function entry/exit). The push and pop of TSan's shadow call stack are inlined instead of calling `__tsan_func_entry/exit`; the trace events are identical.
  - It gains on Redis and on MySQL on AMD.
  - On MySQL on Intel it loses. FE executes up to 8 % fewer instructions per transaction there but no fewer cycles, because the +59 % code size raises i-cache and iTLB stalls. The debug-build MySQL is lock-bound, so longer transactions hold locks longer.
  - *The processor decides the sign, contention the size.* Part of it may be layout (see the note at the top).
- **VWIDE** (idea 7, "cheap run-time DE checks"). Exact removal also at sites that no covering check dominates but that usually hit in practice; an inline hit test verifies the cover at run time. **VWIDE-loops** does this only inside loops. Alone they gain nothing (table 2); here they appear combined with FE.
- **N1**. TSan's fast-path hit test ("is this access already recorded in shadow?") is inlined at every access, and the runtime is called only on a miss. **N1-L**: the same, only in small loops.
- **DynSTC-RT**. A run-time single-thread mode: while the process has one live thread, the runtime records nothing, and the create/join edges order everything else. Its whole gain is FFmpeg's single-threaded stream copy.
- **N1-ST**. N1's inline test is skipped while the thread is in that single-thread mode, where it would always miss.
  - **front**: the flag is read before the test. This gives the largest gain on single-threaded code and a 1-3 % tax on multi-threaded code.
  - **miss**: the flag is read only on a miss. There is no tax on hits and about half the gain.
- **MEMINTR** (idea 8b). A memcpy/memmove whose source is a constant or an uncaptured local has only its destination checked. Small gains on SQLite (checkpoint_starvation_1 +3.8 % in both apollo sessions) and memcached. Redis loses 2.5 %; the cause is under review (two hot interceptor copies, or page effects).
- **exact DE package**. DE's merging of adjacent checks and its loop ranges, made exact under verified removal.

---

## ⚪ Table 2 — optimizations that gain nothing (or lose)

| optimization | SQLite | memcached | Redis | MySQL | FFmpeg | verdict |
|---|---|---|---|---|---|---|
| **LIBCALL-INLINE** (light compare entries) | `a` ⚪ 0.0 | `a` ⚪ +0.4 | `f` ⚪ −0.7 (screening) | `a` ⚪ −0.4 | `f` ⚪ +0.3 | **closed** 28 Sep: correct and exact, no effect |
| **N1-CSE** | `a` ⚪ +0.3 | `a` ⚪ −0.4 · `f` ⚪ −1.0 (screening) | — | `a` ⚪ +0.9 · `f` 🟡 −1.6 (screening) | — | no effect on the three apps it was built for |
| **N1-ATOMIC** | `a` ⚪ −0.4 (without stress2 +0.9; CV tripped, re-run queued) | — | — | — | — | no effect on its target (SQLite's relaxed atomics) |
| **FE-SINK v2** ³ | `a` ⚪ +1.1 (stress2 alone); with FE-INL 🟡 −1.0 | `f` 🟡 −1.6 (screening) | `f` ⚪ +0.7 (screening) | pend. | `f` 🟡 −1.2; with FE-INL 🟡 −1.0 | loses 1-2 % so far; remaining legs today (below) |
| VWIDE (alone) | `a` ⚪ −0.6 | `a` ⚪ +0.6 | `f` ⚪ −0.5 | `f` ⚪ −0.3 · `a` ⚪ −0.3 | `f` ⚪ +0.8 | noise; no effect together with N1 |
| VWIDE-loops (alone) | `a` ⚪ +0.2 | `a` ⚪ 0.0 | `f` ⚪ −0.9 | `f` 🟡 −1.8 | `f` 🟡 +1.3 | noise |
| N1-PM (N1 + preserve_most) | `a` 🔴 −2.8 · `f` 🔴 −2.1 (n=1) | `f` ⚪ −1.0 · `a` ⚪ −0.2 | `f` 🟢 +6.0 `?` (screening; idle re-run queued) | `f` 🔴 −2.2 (screening) · `a` 🔴 −2.5 | `f` 🟢 +5.2 (mjpeg +14) | the FFmpeg gain is N1's own (+5.0); loses on SQLite/MySQL |
| N1-LOOPS-∞ (N1-L without cap) | `a` 🟡 −1.3 · `f` ⚪ −0.4 (n=2) | `f` 🟢 +3.0 `?` (screening; idle re-run queued) · `a` ⚪ −0.8 (2 sessions) | `f` 🟡 +1.9 (screening) | `f` 🟡 −1.8 (screening) · `a` ⚪ −0.1 | `f` 🟢 +3.3 (mjpeg +9) | the FFmpeg gain is N1-L's own (+3.0) |
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

<sub>² Measured over the shipped artifact (tier A), not over P1-v3. ³ FE-SINK is still under conditions: audit A23 is sound with conditions and premise P5 is pending. Its FFmpeg and SQLite cells are readings of record; the rest are screening.</sub>

**🔎 Key to table 2**
- **LIBCALL-INLINE**. Calls from instrumented code to strcmp, strncmp, strcasecmp, memcmp, bcmp, memchr and strlen go to light runtime entries. These record the interceptor's read ranges without its frame.
  - It saves ~3-6 ns per call (microbenchmark, 32 B) against 7-13 ns for the two range checks that both paths pay.
  - At the observed call rates (up to 2.3 M/s) that is ≤ 0.8 % anywhere.
  - Audits A21/A21b: sound with conditions (no LTO; suppression patterns).
- **N1-CSE**. A compact inline hit test: a run of accesses shares the sibling shadow address and the byte mask. It is read over a control built from the same root.
- **N1-ATOMIC**. N1's inline hit test extended to relaxed atomic loads and stores. SQLite's walFindFrame does 151 M of them per run, and 55.5 % of the loads hit.
- **FE-SINK v2**. `__tsan_func_entry` is sunk to the first point that needs the frame, so invocations that only hit never push one. It removes ~22 % of executed entries on FFmpeg and Redis, but that is < 1 % of cycles against 2-12 % more code.
  - Arms today: FB is the control, FI is FE-INL alone, FS is sink with call entries, FIS is sink with FE-INL, and FIC is FE-INL-CSE.
  - Still to run: memcached and MySQL on apollo, Redis on focs.
- **FE-INL exit-max=1**: FE-INL with at most one inlined exit per function, a variant for C++ unwinding.
- **N1b**: the runtime call is kept but goes through cheaper `preserve_most` entry points, which save fewer registers.
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
| ⭐ **LO-OBJ** | lock ownership relative to an object: fields of objects owned by one SQLite BtShared (pages, WAL, Pager), accessed while its mutex is held | design and cheap checks done (unit tests and a mutation test pass); needs ownership annotations taken from SQLite's own `mutex_held` assertions | 🟢 SQLite ≈ ×1.17-1.21 (upper estimate); memcached ≤ +0.2; MySQL ≈ 0 |
| 🆕 **RANGE-UNIFORM** | on a range-check miss, when consecutive shadow cells hold identical values, race-check one cell and write the rest with wide stores | opened 28 Sep; exact against the stock vectorised walk, no premise needed (each cell is still loaded and compared); census of uniform missed cells today | memcached: missed range cells ≈ 18 % of user CPU (4 % of server CPU); the share uniform cells can save is unknown |
| 🆕 **slot / epoch budget** (runtime) | stock TSan's slot preemption chains and global resets | measured 28 Sep, per run: memcached 14 M preemptions and 456 DoResets (each wipes all shadow), MySQL 2.6-2.8 M and ~200, FFmpeg ~150 k and 16, Redis 0. The slot pick is already free-first; the lever is the epoch budget (≈ 1.9 G release-driven increments per memcached run). Cycle-cost census today | memcached slot machinery ≈ 9 % of cycles (earlier profile) |
| **EA-SLOT** (idea 8a) | a value loaded from a stack slot and passed on escapes as the slot's contents, not the slot itself (memcached's `tokens` array) | repaired design v2 found unsound by audit (3 new counterexamples, fixes C11-C13); census branch ready | ≈ 3 % of memcached's checks |
| **CLONE-ESC** (idea 9) | clone functions with hot pointer arguments into escaping and non-escaping versions; choose the clone at run time | timed oracle OA0 (unsound upper bound): SQLite ≤ +2.6 % (screening), memcached ≤ +3.5 %, Redis ≤ +6.0 %, FFmpeg ≤ +10.4 % (the stream-copy part overlaps DynSTC-RT's gain) | ≤ 7 % FFmpeg/Redis, ≤ 3 % SQLite, ≤ 1 % memcached, 0 MySQL |
| 🆕 **FE-INL-CSE** | one thread-state load for a function's entry, exits and inline tests | audit A23 blocker (thread state read before its init) fixed; preservation with it on is clean; timed today as an evidence-only arm in the FE-SINK legs (it relies on the unadopted A3-fiber premise) | small; trims FE's code on MySQL |
| 🆕 **MEMINTR-INLINE** | small constant-size memcpy/memset/memmove kept as intrinsics and checked by range entries or inline tests instead of the interceptor call | census + design | limited by the range checks it keeps (see RANGE-UNIFORM) |
| 🆕 **N1-SPLIT** | N1/VWIDE miss blocks marked cold and split to .text.unlikely | compile-only check first | front-end loss on MySQL-f, SQLite, memcached |
| 🆕 **N1-PAIR** | one 32-byte load + movmsk tests two adjacent shadow cells | census | SQLite page/record parsing |
| 🆕 **SPIN-ACQ** | in a loop of only atomic loads + arithmetic, poll relaxed and acquire only when the value changes (Redis getIOPendingCount: 19.1 G seq_cst acquires) | ceiling only (Alexey); can only add reports (ABA), never lose one | Redis ≤ +14 % (io-threads bound) |
| 🆕 **compile time** | compile-time overhead of P1-v3 AllOpt over stock | measured (quiet, one build at a time): CPU user+sys SQLite +20.1 %, memcached +8.4 %, Redis +9.4 %, FFmpeg +15.6 %; worst unit sql_yacc.cc +55 %; N1 inline ×1.8-3.3; MySQL whole still open | paper says 1-18 % |
| N1-ST (b), (c) | hoist the single-mode flag read per call-free run; or encode single mode in the fast state TSan already loads | not built | would halve or remove N1-ST front's 1-3 % multi-threaded tax |
| RANGE-HIT-SKIP / RANGE-VEC | skip the range trace event when every cell already holds the access / a vectorised hit walk | ❌ closed by census 28 Sep. The all-hit share is FFmpeg 81 %, SQLite 73 %, Redis 50 %, memcached 11 %. Bounds: RANGE-HIT-SKIP ≤ 0.53 % (≤ 1.1 % conservative, SQLite only); RANGE-VEC ≤ 0.9 % | — |
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
| RT-SYNC / RT-ALLOC | runtime-only speedups of the sync path and the allocator | ⏸️ parked: outside the paper's topic (the range and slot parts were re-opened as censuses on 28 Sep, rows above) | — |

**🔎 Key to table 4**
- **Ceiling**: an upper bound from a profile oracle or a census of executed checks, not a timed optimization.
- **Census**: a static or profile count of the checks or runtime events a rule would remove, weighted by how often they run.
- ⏸️ **Parked**: set aside with its data kept. ❌ **Closed**: measured or argued to have no worthwhile gain, or unsound.
- **Audit**: an independent read-only review against the zero-lost-races rule, done before anything is timed.
- ⭐ marks the one direction with large headroom on an app where nothing else helps (SQLite).
</content>
</invoke>
