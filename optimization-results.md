# TSan instrumentation optimizations: results

State: 2 Oct 2026. All figures are speedups over stock TSan unless a table says otherwise. `a` = AMD (2 × EPYC 9115),
`f` = Intel (Xeon w9-3495X). The long version of this file is in the history (commit 3abcd2c).

🟢 gain resolved above the A/A control · 🟡 unresolved, or 1-2 % · ⚪ within ±1 % · 🔴 loss of 2 % or more · — not measured

## Table 1a. All applications, without annotation-based optimizations

| app | workload | stock TSan over native | configuration | AMD | Intel | submitted paper |
|---|---|---|---|---|---|---|
| FFmpeg | four transcodes of one film | 2.8× | DynSTC-RT + N1 + N1-ST | 🟢 **+29.2 %** over upstream TSan (direct, 1.285-1.305, 4 offsets) | 🟢 **+30.5 %** over upstream TSan (direct, 1.290-1.314, 4 offsets); +34.0 % over our stock arm | +57 % |
| Redis | seven data-heavy commands (LRANGE_100/300/500/600, MSET, ZADD, ZPOPMIN) | 6.0× | FE-INL + N1 | 🟢 **+7.4 %** over upstream TSan (direct, 1.055-1.096, 4 offsets; server with 8 I/O threads and client on disjoint CPUs); +10.2 % (1.096-1.109) over our stock arm | 🟢 **+5.7 %** over upstream TSan (direct, 1.043-1.073, 4 offsets; server with 12 I/O threads and client on disjoint CPUs); +8.1 % over our stock arm; +7.6 % on shared CPUs with 20 I/O threads | +45 % |
| MySQL | Release build, sysbench insert, update_non_index, delete; 24 connections, server and client on disjoint CPUs | 7.5× | FE-INL on AMD; the paper's analyses alone on Intel | 🟢 **+9.4 %** over upstream TSan (direct, 4 offsets); +8.0 % (1.067-1.083) over our stock arm; +5.4 % on shared CPUs | ⚪ −0.2 %; with FE-INL 🔴 −3.0 % (0.968-0.971) | +16 % (select), +11 % (write-only) |
| memcached | pipelined 32-key gets with 190-byte keys; server and client on disjoint CPUs | 6.7× | the paper's analyses + EA-CONTENTS + SWMR-ROOTS | 🟡 +1.0 % over our stock arm (1.004-1.015, A/A 0.995-1.002); −1.7 % over upstream TSan (derived) | 🟡 +3.2 % over our stock arm (4 offsets); about +1.2 % over upstream TSan | +7 % |
| SQLite | threadtest3, shared-cache subtests stress2 and create_drop_index_1 | 4.6× and ~19× | the paper's analyses | ⚪ −1.8 % (0.950-1.025, inside the A/A 0.942-0.985; 4 offsets) | — | +71 % |

FFmpeg's figure is the geomean of four transcodes and is carried by the single-threaded stream copy: over upstream,
copy 2.71× (AMD) / 2.88× (Intel), mjpeg +4.2 % / +3.7 %, h264 −0.2 % / −1.7 %, h265 −1.1 % / −1.1 %. The two encoders
run mostly in uninstrumented x264/x265 code (stock TSan costs them 1.3-1.4×), and the inline single-thread test costs
them slightly.

The last column is what the submitted paper printed for all its analyses together: other workloads, the Intel host,
and earlier compilers with two elisions later found unsound.

## Table 1b. Applications where annotation-based optimizations gain

A short per-application description names objects and their owner; the compiler checks it against the code, and a
run-time guard tests the ownership condition. Workloads as in table 1a.

| app | configuration | AMD | Intel |
|---|---|---|---|
| memcached | EVCONF + SWMR-ROOTS + EA-CONTENTS | 🟢 **+30.4 %** over upstream TSan (direct, 1.300-1.307, 2 offsets); +32.2 % (1.300-1.335) over our stock arm | 🟢 **+22 %** over upstream TSan (1.244 over our stock arm in the same leg); +25.4 % (1.245-1.263) in the first leg |
| SQLite | LO-OBJ-G | 🟢 **+18.5 %** (1.155-1.215, 4 offsets × N=4, both subtests: stress2 +16.0 %, create_drop_index_1 +21.2 %; over our stock arm, the faster base here; +23.5 % over upstream TSan) | — |

- One configuration for every app (derived): N1 + N1-ST + DynSTC-RT gives FFmpeg +24 %, Redis +6.5 %, and loses on
  SQLite (−0.5…−3.6 %), memcached (−6.7 %) and MySQL (−2.6 %).
- The paper's analyses alone (P1-v3, below) are not separable from stock on any app (0.98-1.02).
- Ceilings for removing checks (profile oracles: memory touched by one thread / also Eraser-consistent): SQLite
  +16 / +53 %, memcached +6 / +11 %, Redis +17 / +27 %, MySQL −1 / +3 %, FFmpeg +31 / +41 %.

## Table 2. Optimizations that gain

Single-lever columns are over the base P1-v3 (the paper's analyses EA, LO, STC, SWMR and DE with every soundness fix;
a covered check is verified by an inline hit test instead of being removed). The last four rows are measured
within the configurations of record in table 1.

| optimization | what it is | SQLite | memcached | Redis | MySQL | FFmpeg |
|---|---|---|---|---|---|---|
| **DynSTC-RT** | single-thread mode in the runtime: while one thread is alive nothing is recorded, range checks included | `a` ⚪ +0.4 | `a` ⚪ −0.3 | `f` 🔴 −2.2 | `a` ⚪ +0.1 | `f` 🟢 **+12.0** |
| **N1** | TSan's "already recorded?" test is inlined; the runtime is called only on a miss | `a` ⚪ −0.8 | `a` ⚪ +0.3 | `f` 🟡 +2.5 | `a` 🟡 −1.7 | `f` 🟢 **+5.0** |
| **N1-ST** | N1's inline test is skipped while the thread is in single-thread mode (flag read before the test) | `a` ⚪ +0.2 | `a` ⚪ −0.2 | `f` 🟡 −1.4 | `a` 🟡 −1.8 | `f` 🟢 **+16.9** |
| **FE-INL** | the push and pop of TSan's shadow call stack are inlined instead of calling the runtime | `a` ⚪ −0.7 | `a` ⚪ +0.8 | `f` 🟢 **+4.8** | `a` 🟢 **+3.6** | `f` ⚪ 0.0 |
| **FE-INL + VWIDE-loops** | plus run-time verified check removal inside loops | `a` ⚪ +0.1 | `a` ⚪ +0.6 | `f` 🟢 **+7.2** | `a` 🟢 **+4.6** | `f` 🟡 +1.8 |
| **FE-SINK** | the function-entry call is moved to the first point that needs the frame | `a` 🟡 +1.1 | `a` 🟡 −1.7 | `f` 🟢 **+4.1** | `a` 🟢 **+2.8** | `f` 🟡 −1.2 |
| **N1-L** | N1 only in loops with at most 20 checks | `a` ⚪ +0.3 | `a` ⚪ −0.9 | `f` 🔴 −2.5 | `a` ⚪ −0.1 | `f` 🟢 **+3.0** |
| **MEMINTR** | a memcpy from a constant or private source has only its destination checked | `a` 🟡 +0.7…+2.5 | `a` ⚪ +0.9 | `f` 🔴 −2.5 | `a` ⚪ +0.3 | `f` ⚪ 0.0 |
| **EA-CONTENTS** | a pointer read from a container no longer makes the container shared | — | `a` 🟢 **+1.4** (on against off) | — | — | — |
| **SWMR-ROOTS** | no checks on reads of a global whose only write precedes every reader thread | — | `a` 🟢 **+0.9** on top of EVCONF | — | — | — |
| **EVCONF** (annotation) | objects annotated as owned by one thread are unchecked while a run-time guard holds (no idle-timeout thread, no external storage, connection never lent) | — | `a` 🟢 **+22.7** over stock on shared CPUs; +32.2 with the two rows above on disjoint CPUs | — | — | — |
| **LO-OBJ-G** (annotation) | objects annotated as protected by their owner's lock are unchecked while the thread holds that lock | `a` 🟢 **+18.8** over stock | — | — | — | — |

- N1-ST is measured on top of N1 + DynSTC-RT and costs 1-3 % on multi-threaded code. FE-SINK is measured over
  P1-v3 + N1 and adds nothing on top of FE-INL.
- FFmpeg, one leg over stock: the paper's compile-time DynSTC +10.1 %, DynSTC-RT +12.2 %, both together +31.9 %,
  the configuration of record +33.2 %. The inline guard skips plain accesses, the runtime mode skips range checks;
  each is about half of the single-threaded work.
- Re-timed on the workloads of record (2 Oct): no other lever adds to memcached's or Redis's configuration.

## Table 3. No gain

| idea | what it is | result |
|---|---|---|
| Removal-mode DE, DE-3R, DE-2R | covered checks deleted outright; one range check per loop; adjacent fields merged | ⚪ MySQL +0.3 %, SQLite 0, memcached 0; 🔴 Redis −3.8 %; FFmpeg open (table 4) |
| DE-5…DE-8 | finer rules for when a call or a cycle breaks a cover | ⚪ −0.8…+1.4 % |
| IPA-DE | a check in a callee covers one in its caller | ⚪ ≤ 1 % of checks |
| VWIDE, VWIDE-loops alone | run-time verified removal at sites no check dominates | ⚪ −0.9…+1.3 % |
| FE-hot, hot-list N1, N1-S, N1-LOOPS-∞ | FE-INL or N1 only at hot sites, by profile or statically | 🔴 none beats the full version |
| FE-PM, N1-PM, N1b | out-of-line entries that save fewer registers | 🔴 MySQL −2 %, SQLite −2…−4 % |
| FE-LAZY, FE-INL-CSE | entry recorded only when needed; one thread-state load per function | ⚪ ±1 % |
| N1-CSE, N1-ATOMIC, LIBCALL-INLINE, N2 | compact, atomic, libc-call and batched variants of the check | ⚪ ±1 % (N2 up to −2 %); N1-ATOMIC on Redis +1.1 % on Intel and +1.6 % on AMD (0.994-1.046 over 4 offsets), not resolved |
| SUBS | a covering record of the same thread counts as a hit | 🔴 −2…−18 %; the selective form is unsound |
| Whole-program summaries alone, STC-TS, allowlists | closed-world facts for STC and SWMR | ⚪ < 1 % of checks |
| LTO, Attributor, extra LLVM passes | more optimization before instrumentation | ⚪ LTO adds nothing to our analyses; the Attributor miscompiles; the passes undo most of the analyses' check removal on Redis |
| CLONE-ESC, EA-SLOT, ICALL-A2, returns-fresh, field chase | finer escape analysis | ⚪ each < 2 % of checks |
| Whole-program mode for MySQL; top-down parameter facts across units | summaries for MySQL's 1,892 units; "this argument is local in every caller" passed to the callee's unit | ⚪ MySQL 0.14 % of checks; the cross-unit facts ≤ 0.14 % on every app |
| Per-field and heap SWMR, thread roles, thread ids by creation history | finer may-happen-in-parallel facts | ⚪ each < 3 % of checks |
| NOALIAS, custom lock wrappers, MySQL sysvars, InnoDB latches, ODR trust | language and library facts | ⚪ each < 3.2 % of checks |
| RT-SYNC, SLOT-CHURN, RANGE-OVERWRITE | runtime-only changes | ⚪ no effect, or below their gate |
| LO-F, LO-W, LO-B1/B2, sound loop ranges, DE-1/2/4/9 | further lock-ownership and DE rules | ⚪ ≤ 1.4 % of checks, or unsound |

## Table 4. Open

| item | what it is | app | status |
|---|---|---|---|
| Redis phase guard (quiet threads) | the main thread, which runs ≥ 99 % of the checks, skips them while every other thread is quiet since a release it acquired; a run-time mode, with an automatic variant and one with a short annotation for signal handlers | Redis | unsound ceiling +45 % over stock; design reviewed (A43), runtime core and compiler half built, tests 15/17; gated build due 6 Oct, then audit and legs |
| Removal-mode DE on FFmpeg | as in table 3 | FFmpeg | +10.2 % over N1 + DynSTC-RT in a screening (mjpeg +29 %); needs a ruling on the report loss below |
| DE "checked on every path", cycle cut | a cover need not dominate if every path has one | all | audited; ≤ 1.5 % of checks; timing queued |
| LO-OBJ-G with the latch (spec v7) | the unlocked page copy closed at run time | SQLite | audited; the figure of record above is v5 with spec v6 (the earlier v4 leg read +16.9 %, and +17.4 % in the same leg); the leg with the latch is queued on the integration compiler |
| MySQL on Intel | FE-INL on the Intel host | MySQL | re-checked 2 Oct with server and client on disjoint CPUs and 24 connections, 4 offsets, A/A 0.994-1.000: FE-INL −3.0 %, the analyses alone −0.2 %. The earlier −6…−12 % came from 36 connections on a 48-CPU set; at 24 connections the layout does not matter (shared CPUs: −3.3 %). On the five other scripts only write_only loses (−5.5 %); reads are neutral |
| Stock control | each compiler's stock arm against upstream TSan with the same checks (fork point plus upstream's capture fix), which isolates the cost of our runtime additions | all | Intel: upstream is faster by 2.5 % on FFmpeg (1.026 / 1.025) and 2.6 % on memcached (1.036 / 1.016), both beyond the A/A, so those figures are re-based; Redis 1.5 % (0.993 / 1.038), not resolved, stands. AMD: memcached 2.7 % (1.031 / 1.023), re-based; MySQL the other way, upstream 1.5 % slower than our stock arm (0.980-0.988), and the integration compiler's stock arm equals upstream there. Direct legs of each best configuration over upstream stock will replace the derived figures |
| Levers re-timed, loop guard | earlier levers on the workloads of record (done on memcached, Redis, MySQL: none adds); the loop guard on non-FFmpeg apps | SQLite, others | SQLite queued; the loop guard runs only on its older compiler, since the camera-ready compiler rejects the flag |
| The old "same location" rule, all combinations | unsound reference: struct fields cover each other, array elements cover each other, both, and a narrower check covers a wider one; counts, speed and lost races | all | requested 2 Oct; being built |
| FFmpeg on AMD | the configuration of record on the AMD host | FFmpeg | running (needed a cpuset delegation on that host) |
| Camera-ready legs | each app's best configuration on the camera-ready compiler, over upstream TSan in the same leg, 4 offsets | all | trees built; held until every hypothesis is checked |
| Integration compiler | every lever in one compiler, behind flags; runtime with upstream's stock-path instruction count | all | gated and audited (A42, A44); cleared as the camera-ready compiler |

Not adopted or parked: OWN-HANDOFF (Redis +13.9 % over P1-v3, but it trusts the program's own thread protocol),
OWN-CONN, DD-EXACT (deadlock detector table: memcached +19.4 %, unsound as committed), TLS-rooted escape analysis
(Debug build only), per-element locksets (1.7 % of memcached's checks), SPIN-ACQ, SWMR-H.

## Table 5. Where the analyses are conservative (estimates)

How much each analysis leaves on the table because it cannot prove an alias or ownership fact that holds at run
time. Shares are of executed checks unless marked; "pending" items are being counted on the workloads of record.

| analysis | what it cannot prove | SQLite | memcached | Redis | MySQL | FFmpeg |
|---|---|---|---|---|---|---|
| DE, dominance | two checks of one invocation hit the same address with no synchronisation in between and the earlier dominates the later, yet both stay checked (share of all executed checks, 3 Oct, record workloads, P1-v3). Includes pairs whose access kind or size cannot cover (a read before a write, a narrow check before a wide one); the split into missing address proofs and non-covering kinds is pending | 3.35 % | 0.04 % | 3.44 % | pending | 5.92 % |
| DE, post-dominance | the same, the later check post-dominating the earlier one, a call between them breaking it / ceiling with calls allowed | 0.28 % / 0.37 % | 0.00 % / 0.00 % | 0.30 % / 0.92 % | pending | 0.21 % / 0.53 % |
| DE, availability | the same pairs where neither check dominates or post-dominates the other (a flag set by the first check would be needed) | 4.36 % | 0.08 % | 5.79 % | pending | 8.56 % |
| DE, "checked on every path" | a cover on every path, none dominating (built, audited) | 0.22 % | 0.00 % | 0.82 % | 0.07 % | 0.5 % |
| DE, cycle cut | a cover lost to a path around a loop (built) | 0.92 % | 0.71 % | 0.14 % | 0.08 % | 0.89 % |
| DE, stronger alias analysis | must-alias from SCEV or points-to analyses | 0 | 0 | 0 | — | 0 |
| EA, all | checks on memory that only one thread touches or that is consistently ordered in the run, and that EA keeps | 72 % | 55 % | 63 % | 36 % never shared, 13 % written before readers | 70 % |
| EA, pointer parameter | the object is reached through a pointer argument, so the callee cannot tell it is local (share of the row above; MySQL: of all checks) | 53-75 % | 53-75 % | 53-75 % | 76 % | 53-75 % |
| EA, cross-unit parameter facts | "this argument is local in every caller", passed to the callee's unit: the upper bound of what it removes | 0.04 % | 0.05 % | 0.00 % | 0.14 % | 0.01 % |
| EA, own stack | the address lies in the accessing thread's own stack and no other thread touches it | 4.9 % | 5.9 % | 12.5 % | 14.8-24 % | 5.8 % |

- The gap between the EA rows is the point: most of the single-thread mass is real at run time but not provable
  statically, because the objects are heap memory reachable from shared structures or passed through callers that
  also pass shared objects. A run-time own-stack test recovers +10 % on MySQL when it skips 34 % of the checks, but
  the test costs 4.5 % and a sound form reaches a third of that, so it is parked.
- **The old "same location" rule.** Before the first soundness review DE treated two accesses to the same object as
  the same location whatever the field, index or size; DE's share of statically removed checks was 32.5 % and is
  2.9 % with the exact rule. That step was 75-100 % of each application's loss of speedup (SQLite ×1.48-1.51 →
  ×1.09-1.10, Redis ×1.41 → ×1.10, memcached ×1.11 → ×1.02, FFmpeg ×1.356 → ×1.018). Being measured now as an
  unsound reference, in every combination: struct fields covering each other, array elements covering each other,
  both, and a narrower check covering a wider access, with executed-check counts, the speedup of each combination
  and the number of stock's races each loses.

## Notes

- **Method.** Each arm is built at four code offsets (0/16/32/48 bytes mod 64) and scored by the mean of per-offset
  ratios; each leg has an A/A arm, and a result counts only if it is above the A/A range at every offset. Screenings
  use two offsets. On memcached the CPU layout changes the size of the effect, so each figure names its layout.
- **Stock arm.** Stock arms are built by each project compiler with every pass off and link its runtime, which
  does extra work even then. Each is timed against upstream TSan with the same race-detection behaviour (the fork
  point plus upstream's fix for unchecked fields of escaped locals); where upstream is faster beyond the A/A at both
  offsets, the figure is re-based by that ratio and marked "derived" until a direct leg over upstream stock exists.
- **memcached's other workloads, AMD, directly over upstream TSan** (disjoint CPUs, 2 offsets): default input
  +2.8 %, pipelining +3.5 %, 32-key gets +6.3 %, long keys +16.8 %, the chosen mix +30.4 %.
- **Removing the runtime's cost.** A runtime that drops the extra tests runs FFmpeg's stream copy and mjpeg exactly as
  fast as upstream (1.031 against upstream's 1.030 over our stock arm); the shippable form on the integration
  compiler recovers three quarters of the gap (1.023). Audit pending.
- **Where the runtime's cost is.** With the same application objects, the project's runtime executes 1-3.5 % more
  instructions than upstream's on the stock path (FFmpeg copy +1.0 %, mjpeg +3.5 %, memcached +2.9 %), on the miss
  and eviction paths of the access entries; layout accounts for about 1 % on FFmpeg's stream copy only.
- **Race preservation.** Every row of tables 1-2 loses no race stock TSan reports under the premises listed in
  `soundness-fixes.md`, checked by IR tests, check-tsan, reproducers with controls and an independent audit.
- **Open soundness points.** Removal-mode DE: after a race report on a cell, a covered write is not re-recorded
  (stock 2 reports, removal 1); a ruling is pending. LO-OBJ-G up to spec v6 missed a race with `sqlite3_serialize`'s
  unlocked page copy, which the measured tests never call; spec v7's run-time latch closes it.
- **MySQL server deaths.** 2 in 104 runs of arms with N1, 0 in 342 others; four checks of N1 found nothing.
- **Compile time** of the paper's analyses over stock (CPU): SQLite +20 %, memcached +8 %, Redis +9 %, FFmpeg +16 %.
