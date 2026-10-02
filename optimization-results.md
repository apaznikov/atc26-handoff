# TSan instrumentation optimizations: results

State: 2 Oct 2026. All figures are speedups over stock TSan unless a table says otherwise. `a` = AMD (2 × EPYC 9115),
`f` = Intel (Xeon w9-3495X). The long version of this file is in the history (commit 3abcd2c).

🟢 gain resolved above the A/A control · 🟡 unresolved, or 1-2 % · ⚪ within ±1 % · 🔴 loss of 2 % or more · — not measured

## Table 1a. All applications, without annotation-based optimizations

| app | workload | stock TSan over native | configuration | AMD | Intel | submitted paper |
|---|---|---|---|---|---|---|
| FFmpeg | four transcodes of one film | 2.8× | DynSTC-RT + N1 + N1-ST | — | 🟢 **+30.7 %** over upstream TSan (derived); +34.0 % (1.329-1.353) over our stock arm | +57 % |
| Redis | seven data-heavy commands (LRANGE_100/300/500/600, MSET, ZADD, ZPOPMIN) | 6.0× | FE-INL + N1 | 🟢 **+10.2 %** (1.096-1.108; 2 offsets, server with 8 I/O threads and client on disjoint CPUs) | 🟢 **+7.6 %** (1.061-1.094; 20 I/O threads) | +45 % |
| MySQL | Release build, sysbench insert, update_non_index, delete | 7.5× | FE-INL on AMD; the paper's analyses alone on Intel | 🟢 **+5.4 %** (1.043-1.060) | ⚪ −0.2 %; with FE-INL 🔴 −3.0 % (0.968-0.971) | +16 % (select), +11 % (write-only) |
| memcached | pipelined 32-key gets with 190-byte keys; server and client on disjoint CPUs | 6.7× | the paper's analyses + EA-CONTENTS + SWMR-ROOTS | 🟡 +1.0 % over our stock arm (1.004-1.015, A/A 0.995-1.002); −1.7 % over upstream TSan (derived) | 🟡 +1.9 % over our stock arm (1.004-1.028, A/A 1.001-1.012), not resolved | +7 % |
| SQLite | threadtest3, shared-cache subtests stress2 and create_drop_index_1 | 4.6× and ~19× | the paper's analyses | ⚪ −0.5 % on an earlier four-subtest set; this set not measured yet | — | +71 % |

The last column is what the submitted paper printed for all its analyses together: other workloads, the Intel host,
and earlier compilers with two elisions later found unsound.

## Table 1b. Applications where annotation-based optimizations gain

A short per-application description names objects and their owner; the compiler checks it against the code, and a
run-time guard tests the ownership condition. Workloads as in table 1a.

| app | configuration | AMD | Intel |
|---|---|---|---|
| memcached | EVCONF + SWMR-ROOTS + EA-CONTENTS | 🟢 **+28.7 %** over upstream TSan (derived); +32.2 % (1.300-1.335) over our stock arm | 🟢 **+22.2 %** over upstream TSan (derived); +25.4 % (1.245-1.263) over our stock arm |
| SQLite | LO-OBJ-G | 🟢 **+18.8 %** (1.140-1.225) | — |

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
| N1-CSE, N1-ATOMIC, LIBCALL-INLINE, N2 | compact, atomic, libc-call and batched variants of the check | ⚪ ±1 % (N2 up to −2 %) |
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
| Redis phase guard | the main thread, which runs ≥ 99 % of the checks, skips them while the I/O threads are parked | Redis | unsound ceiling +45 % over stock; design under review, in work |
| Removal-mode DE on FFmpeg | as in table 3 | FFmpeg | +10.2 % over N1 + DynSTC-RT in a screening (mjpeg +29 %); needs a ruling on the report loss below |
| DE "checked on every path", cycle cut | a cover need not dominate if every path has one | all | audited; ≤ 1.5 % of checks; timing queued |
| LO-OBJ-G with the latch (spec v7) | the unlocked page copy closed at run time | SQLite | audited; the figure of record above is v5 with spec v6 (the earlier v4 leg read +16.9 %, and +17.4 % in the same leg); the leg with the latch is queued on the integration compiler |
| MySQL on Intel | FE-INL on the Intel host | MySQL | re-checked 2 Oct with server and client on disjoint CPUs and 24 connections, 4 offsets, A/A 0.994-1.000: FE-INL −3.0 %, the analyses alone −0.2 %. The earlier −6…−12 % came mostly from 36 connections on shared CPUs. A diagnostic in the AMD arrangement and the five other scripts are queued |
| Stock control | each compiler's stock arm against upstream TSan with the same checks (fork point plus upstream's capture fix), which isolates the cost of our runtime additions | all | Intel: upstream is faster by 2.5 % on FFmpeg (1.026 / 1.025) and 2.6 % on memcached (1.036 / 1.016), both beyond the A/A, so those figures are re-based; Redis 1.5 % (0.993 / 1.038), not resolved, stands. AMD: memcached 2.7 % (1.031 / 1.023), re-based; MySQL is queued. Direct legs of each best configuration over upstream stock will replace the derived figures |
| N1-ATOMIC on Redis | the inline hit test for atomic operations, on top of FE-INL + N1 | Redis | AMD screening: +2.8 % over the best (1.011-1.046, A/A 0.995-1.004); the 4-offset leg is queued |
| Levers re-timed, loop guard | earlier levers on the workloads of record | MySQL, SQLite | queued |
| Integration compiler | every lever in one compiler, behind flags | all | gated and audited; identity checks running |

Not adopted or parked: OWN-HANDOFF (Redis +13.9 % over P1-v3, but it trusts the program's own thread protocol),
OWN-CONN, DD-EXACT (deadlock detector table: memcached +19.4 %, unsound as committed), TLS-rooted escape analysis
(Debug build only), per-element locksets (1.7 % of memcached's checks), SPIN-ACQ, SWMR-H.

## Notes

- **Method.** Each arm is built at four code offsets (0/16/32/48 bytes mod 64) and scored by the mean of per-offset
  ratios; each leg has an A/A arm, and a result counts only if it is above the A/A range at every offset. Screenings
  use two offsets. On memcached the CPU layout changes the size of the effect, so each figure names its layout.
- **Stock arm.** Stock arms are built by each project compiler with every pass off and link its runtime, which
  does extra work even then. Each is timed against upstream TSan with the same race-detection behaviour (the fork
  point plus upstream's fix for unchecked fields of escaped locals); where upstream is faster beyond the A/A at both
  offsets, the figure is re-based by that ratio and marked "derived" until a direct leg over upstream stock exists.
- **Race preservation.** Every row of tables 1-2 loses no race stock TSan reports under the premises listed in
  `soundness-fixes.md`, checked by IR tests, check-tsan, reproducers with controls and an independent audit.
- **Open soundness points.** Removal-mode DE: after a race report on a cell, a covered write is not re-recorded
  (stock 2 reports, removal 1); a ruling is pending. LO-OBJ-G up to spec v6 missed a race with `sqlite3_serialize`'s
  unlocked page copy, which the measured tests never call; spec v7's run-time latch closes it.
- **MySQL server deaths.** 2 in 104 runs of arms with N1, 0 in 342 others; four checks of N1 found nothing.
- **Compile time** of the paper's analyses over stock (CPU): SQLite +20 %, memcached +8 %, Redis +9 %, FFmpeg +16 %.
