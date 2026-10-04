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
| memcached | EVCONF + SWMR-ROOTS + EA-CONTENTS (+ EVCONF-RANGES) | 🟢 **+30.4 %** over upstream TSan (direct, 1.300-1.307, 2 offsets); +32.2 % (1.300-1.335) over our stock arm. **With EVCONF-RANGES (4 Oct, V4, 2 offsets): +18.5 % on top; with EVCONF-ARGS too, +7.3 % more; about +66 % over upstream (derived: 1.304 x 1.185 x 1.073)**; on the V3 shape the two read +10.3 % and +5.1 % | 🟢 **+22 %** over upstream TSan (1.244 over our stock arm in the same leg); +25.4 % (1.245-1.263) in the first leg. **With EVCONF-RANGES (4 Oct): +13.9 % on top (1.127-1.155, 4 offsets), about +41 % over our stock root (derived); with EVCONF-ARGS too, +7.2 % more, about +51 % (derived)**, pending the in-bounds premise |
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
| **EVCONF-RANGES** (annotation) | EVCONF's guard applied to memset/memcpy of constant length on an owned object; one memset zeroing each response object was 16.6 % of the cycles | — | `f` 🟢 **+13.9**, `a` 🟢 **+18.5** (V4; +10.3 on V3) over the configuration of record (Intel equals the unsound ceiling, +14.1; root alone −1.3) | — | — | — |
| **EVCONF-ARGS** (annotation) | EVCONF's confinement carried into the hash function's key argument: a clone of MurmurHash3 with the key reads unchecked while the guard holds, called only where the key is a covered request key | — | `f` 🟢 **+7.2**, `a` 🟢 **+7.3** (V4; +5.1 on V3) over the camera-ready configuration (with EVCONF-RANGES); the unsound ceiling of all key reads was +8.2 (Intel) | — | — | — |
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
| DE "checked on every path", cycle cut | a check is covered if every path to it has a cover, even when none dominates; covers kept around loops | ⚪ over the best, AMD, 4 offsets: Redis +1.0 %, memcached −1.2 % (Intel), SQLite −0.6 %, MySQL +0.1 %, all inside their A/A (3 Oct); ≤ 1.5 % of checks |
| Loop guard (T8) and the T10 package | loop guard: a loop-invariant access is checked once per synchronisation-free stretch and re-checked when the shadow generation moves; T10: the older compiler's full lever package (ALL, with N1-L) including it | ⚪ over the paper's analyses, 4 offsets (4 Oct): Redis +2.2 / +3.1 %, memcached 0 / +1.6 %, SQLite not resolvable (A/A ±7 %), MySQL −0.1 / 🔴 −4.2 %; no camera-ready arm |
| Earlier levers on SQLite v7 | N1, N1-L, MEMINTR, FE-INL on top of the configuration of record | ⚪ FE-INL +4.5 %, MEMINTR +3.5 % at the edge of a wide A/A (0.974-1.053), N1 🔴 −18.5 %; FE-INL and FE-INL + MEMINTR are arms of SQLite's camera-ready leg |
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
| Redis phase guard (quiet threads) | the main thread, which runs ≥ 99 % of the checks, skips its checks (range checks included) while every other thread is quiet since a release it acquired; an automatic variant, and an annotated one that also attests the background threads' start | Redis | **measured, not yet quotable (4 Oct, Intel, on the audited tip, offsets 16/32/48, A/A 0.991-1.006): annotated +26.7 % over the best configuration, about +34 % over upstream TSan in the same leg (1.267 / 0.947)**; AMD (4 offsets, A/A 0.946-1.004): +17.8 % over the best configuration (1.187-1.209 at offsets 16/32/48), about +22 % over upstream (1.178 / 0.965); the timed trees equal the audited tip's apart from one compare bound in config.c; those legs ran with the range skip off. **With the range skip on (fixed, audited A56; final root 2edcfa3b31d6), Intel: 1.314 over the best configuration (1.303-1.331, 4 offsets, A/A 0.988-1.006), +9.9 % over the same arm with ranges off; about +39 % over upstream (derived with the 4 Oct control ratio 0.947)**; AMD leg next; with fewer I/O threads (4 and 6 instead of 12, ranges off) the annotated arm reads 1.277 and 1.265, so the gain does not depend on that setting; a paired leg of the old and final roots (5 Oct) shows them equal (bests 1.001, annotated arms 0.998) and reads the ranges arm at 1.274 (+4.6 % over ranges off): between legs the figures move by about ±3 %, more than the within-leg A/A, so the quoted Redis numbers will pool several legs (N=2 per offset); the automatic variant never enters quiet mode on this workload (Redis has no global reset in a run) and costs 3.8 %; about 6 % of fixed cost is compiled code (longer hot functions), being reduced. Audits A45-A51 with fixes; quotable after the last fixes, audit A52 and a ruling on the Redis config-table premise; AMD leg next |
| Removal-mode DE on FFmpeg | as in table 3 | FFmpeg | +10.2 % over N1 + DynSTC-RT in a screening (mjpeg +29 %); the report-loss premise (P-REPORT) was adopted 3 Oct, so it is admissible; not pursued, since FFmpeg's configuration is frozen (3 Oct) |
| EVCONF-RANGES | as in table 2 | memcached | audited (A46, A46b, A46c: sound under the in-bounds premise), gated; Intel +13.9 %, AMD +10.3 % (table 2); camera-ready root tsan-evr-a94e05a7e778 gated; the in-bounds premise awaits a ruling |
| EVCONF-ARGS and memcached's flag globals | ceilings on top of memcached's camera-ready configuration: the request key's reads in murmur3 (EVCONF's confinement carried into the hash function's pointer argument), then also the unlocked reads of `settings`, `expanding` and `hashpower` | memcached | unsound ceilings over the camera-ready tree: key reads Intel +8.2 %, AMD +5.7 %; both Intel +13.0 %, AMD +8.4 % (A/A 0.979-0.999 / 1.001-1.020). EVCONF-ARGS reaches ~83 % of the key reads (request keys, not item keys): design and code audited (A50, A54: sound with conditions), per-copy control S = 0; Intel +7.2 %, AMD +5.1 % (table 2); part of memcached's camera-ready configuration (root a94e05a7e778 + f1af4f1d1531). The flag globals have no sound route (admin-written settings; `expanding` is a genuine benign race), closed |
| DE-AV | DE's covers checked at run time where no dominance holds (a flag set by the first check) or where only the address equality is unproven | SQLite, Redis, MySQL | unsound ceilings, SQLite on AMD (2 offsets, A/A 0.975-1.013): availability +9.1 %, address question +3.4 %; repeat at N=6 (A/A 0.999-1.019): availability +10.8 % on both offsets, address nil; Intel, 4 offsets (A/A 0.999-1.013): availability +5.3 %, address +1.5 % (noise); AMD at 4 offsets (A/A 0.953-1.060): availability +8.0 %. **Parked 4 Oct:** a run-time flag (the thread's TSan state word, optionally with the address) reaches 3-17 % of the executions at a test cost of 0.2-0.3 checks, so the expected gain is about −2…+4.5 % on AMD and half that on Intel; a prototype would settle it, after the higher items; Redis on Intel: +0.1 % / −0.9 %, nil (A/A 0.985-1.006); MySQL after its lists |
| LO-OBJ-G on the connection's objects | SQLite's VDBE under construction treated as owned by `db->mutex` | SQLite | unsound ceiling +5.8 % on AMD (A/A 0.938-1.030); needs two lock types, a second owner slot and a ruling on how the lock is asserted |
| LO-OBJ-G with the latch (spec v7) | the unlocked page copy closed at run time | SQLite | audited; the figure of record above is v5 with spec v6 (the earlier v4 leg read +16.9 %, and +17.4 % in the same leg); the leg with the latch is queued on the integration compiler |
| MySQL on Intel | FE-INL on the Intel host | MySQL | re-checked 2 Oct with server and client on disjoint CPUs and 24 connections, 4 offsets, A/A 0.994-1.000: FE-INL −3.0 %, the analyses alone −0.2 %. The earlier −6…−12 % came from 36 connections on a 48-CPU set; at 24 connections the layout does not matter (shared CPUs: −3.3 %). On the five other scripts only write_only loses (−5.5 %); reads are neutral |
| Stock control | each compiler's stock arm against upstream TSan with the same checks (fork point plus upstream's capture fix), which isolates the cost of our runtime additions | all | Intel: upstream is faster by 2.5 % on FFmpeg (1.026 / 1.025) and 2.6 % on memcached (1.036 / 1.016), both beyond the A/A, so those figures are re-based; Redis 1.5 % (0.993 / 1.038), not resolved, stands. AMD: memcached 2.7 % (1.031 / 1.023), re-based; MySQL the other way, upstream 1.5 % slower than our stock arm (0.980-0.988), and the integration compiler's stock arm equals upstream there. Direct legs of each best configuration over upstream stock will replace the derived figures |
| The old "same location" rule, all combinations | unsound reference: struct fields cover each other, array elements cover each other, both, and a narrower check covers a wider one; counts, speed and lost races | all | done 4 Oct (Table 5 notes): S + A buys 6-12 % (SQLite 27 %) over exact DE; S loses a real memcached race; nothing adoptable; kept as the related-work bound |
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
| DE, dominance | two checks of one invocation hit the same address with no acquire between them at run time and the earlier dominates the later, yet both stay checked (share of all executed checks, 3 Oct, record workloads, P1-v3) | 3.35 % | 0.04 % | 3.44 % | 5.45 % | 5.92 % |
| of which: a call on some path between them | DE refuses unless the callee is proven free of synchronisation; a call to an external or indirect callee / to a local or inlined one | 1.67 % / 0.69 % | 0.04 % / 0 | 3.38 % / 0.00 % | 2.78 % / 1.06 % | 3.53 % / 0.57 % |
| of which: the address question | no call between, only the equality of the two addresses is unproven (array elements with equal indexes, two loads of one field, a loop phi) | 1.0-1.7 % | 0 | 0.06 % | 1.6-2.7 % | 1.8-2.4 % |
| DE, post-dominance | the same, the later check post-dominating the earlier one, a call between them breaking it / ceiling with calls allowed | 0.28 % / 0.37 % | 0.00 % / 0.00 % | 0.30 % / 0.92 % | 0.02 % / 0.53 % | 0.21 % / 0.53 % |
| DE, availability | the same pairs where neither check dominates or post-dominates the other (a flag set by the first check would be needed) | 4.36 % | 0.08 % | 5.79 % | 1.00 % | 8.56 % |
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
- **The old "same location" rule, measured as an unsound reference (3 Oct).** Before the first soundness review DE
  treated two accesses to one object as the same location whatever the field, index or size; that step was 75-100 %
  of each application's loss of speedup. Rebuilt behind three flags: S (fields of one struct cover each other),
  A (elements of one array cover each other; any array indexing makes a pair an array pair), Z (a narrower check
  covers a wider access). Executed checks removed beyond today's exact DE, on the paper's analyses in removal mode:

  | combination | SQLite | memcached | Redis | MySQL | FFmpeg |
  |---|---|---|---|---|---|
  | S | 10.7 % | 9.5 % | 21.1 % | 13.5 % | 13.7 % |
  | A | 9.3 % | 9.0 % | 8.8 % | 4.1 % | 22.1 % |
  | Z | 0.0 % | 0.0 % | 0.0 % | 0.0 % | 0.0 % |
  | S + Z | 13.0 % | 11.0 % | 23.0 % | 14.1 % | 14.1 % |
  | S + A | 19.7 % | 17.5 % | 29.8 % | 17.5 % | 34.4 % |
  | S + A + Z | 22.1 % | 20.3 % | 32.0 % | 18.2 % | 35.4 % |

  Most relaxed removals are dominance covers (post-dominance adds 7-20 % of them). Fields and elements are told apart
  by type-based alias metadata, so S and A are a heuristic split. Lost races (3 Oct, stock 5 runs, each arm 2): SQLite loses
  none in any arm; on memcached A and exact DE lose none, S and S + A + Z lose 2 sites stock reports in every run
  (`clock_handler` reads `stats_state.curr_items` without the stats lock, while `do_item_link` and `do_item_unlink`
  write `curr_bytes` and then `curr_items` under it: under S the `curr_bytes` write covers the `curr_items` write,
  so a genuine race goes unreported). Speed over exact DE, removal mode, Redis on Intel (3 Oct, A/A 0.989-1.009):
  S +8.4 %, A +5.9 %, S + A +11.5 %, S + A + Z +10.6 %; SQLite on AMD (A/A 0.997-1.049): S +11.9 %, A +12.8 %,
  S + A +27.3 %, S + A + Z +31.2 % (create_drop_index_1 +45.5 %, stress2 +18.4 %); memcached on Intel (A/A
  0.997-1.010): S +1.7 %, A +3.9 %, S + A +6.6 %, S + A + Z +6.8 %; MySQL on AMD (A/A 0.988): S +4.1 %, A +1.1 %, S + A +6.0 %,
  S + A + Z +7.9 %. The shadow-proxy rule of RedCard, the core of S that keeps "at least one race is reported where stock
  reports one", was counted and not adopted, since it reports a different race than stock: it would remove only
  memcached 0.13 %, Redis 1.06 %, SQLite 0.77 %, FFmpeg 0.11 % of executed checks, against S's 9.5-22.5 %, because most
  fields are touched by a memory intrinsic or lack one proxy that accompanies every access.

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
  `soundness-fixes.md`, checked by IR tests, check-tsan, reproducers with controls and an independent audit. On the
  camera-ready compiler (3 Oct), application runs against stock (10 runs per arm, races matched by location pair)
  lost no race in any app, for the best configuration and for best + DE all-paths and cycle cut: memcached 4 races
  kept (9 more appear in only 1 of 10 stock runs), SQLite 3 kept, MySQL 232 and 236 kept over 5 sysbench scripts,
  Redis (no race on its benchmark) all 10 seeded races kept, including three placed where the DE covers fire.
  FFmpeg's stock reports no race, so it certifies nothing.
  The final memcached configuration (EVCONF + RANGES + ARGS, 4 Oct) against stock on its own root, 10 runs: 4 races
  kept, 0 lost, also with DE all-paths and cycle cut; check-tsan 12/12 configurations pass (390 tests each), go-check
  passes. Redis's quiet mode is next (seeded races, ranges on and off).
- **Open soundness points.** Removal-mode DE: after a race report on a cell, a covered write is not re-recorded
  (stock 2 reports, removal 1); a ruling is pending. LO-OBJ-G up to spec v6 missed a race with `sqlite3_serialize`'s
  unlocked page copy, which the measured tests never call; spec v7's run-time latch closes it.
- **MySQL server deaths (closed 4 Oct).** 2 in 104 runs of arms with N1, 0 in 342 others, on 27-28 Sep Debug roots: one
  lost connection, one InnoDB debug assertion. Neither root is an ancestor of today's N1 code (their distinctive N1
  changes, a miss-path single skip and N1-CSE load sharing, are not in it). A sweep of every N1 run since (Redis 715,
  FFmpeg 660, memcached 258, SQLite 184, MySQL 148) finds no other N1 failure; MySQL since then: 0 deaths in 670 runs.
- **Compile time** of the paper's analyses over stock (CPU): SQLite +20 %, memcached +8 %, Redis +9 %, FFmpeg +16 %.
