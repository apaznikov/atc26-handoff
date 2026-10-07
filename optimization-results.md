# TSan instrumentation optimizations: results

State: 7 Oct 2026. Longer earlier versions: 90a735e, f02509d, b8d5c56. Premises: `soundness-fixes.md` §6.

## Table 1. Summary (AMD, 2 × EPYC 9115)

Speedup = stock TSan's run time over the optimized build's, same leg, geometric mean over 4 code offsets, stock built
by the same compiler and runtime. Every figure is the best sound one measured (no race stock TSan reports is lost,
under the premises of §6).

| app | workload | submitted paper | (1) no annotations, static | (2) no annotations, run-time checked | (3) spec generated from the program's assertions | (4) our annotations |
|---|---|---|---|---|---|---|
| SQLite | threadtest3: stress2, create_drop_index_1 | 1.71× | 1.02× (the paper's analyses) | — | **1.16×** (LO-OBJ-G, gen6 = gen8's output; run-time lock guard) | **1.25×** (LO-OBJ-G, spec v7; run-time guarded) |
| FFmpeg | four transcodes of one film | 1.57× | — | **1.29×** (DynSTC-RT + N1 + N1-ST) | — | — |
| Redis | 7 data-heavy commands, 8 I/O threads | 1.45× | 1.10× (FE-INL + N1) | **1.54×** (FE-INL + N1 + quiet threads derived from the whole program) | — | 1.40× (hand-written quiet-thread lines; superseded by (2)) |
| MySQL | Release, sysbench insert / update / delete, 24 connections | 1.16× | **1.13×** (FE-INL) | — | — | — |
| memcached | pipelined 32-key gets, 190-byte keys (V4) | 1.07× | **1.02×** (the analyses + EA-CONTENTS + SWMR-ROOTS + FE-INL + MEMINTR) | **1.34×** (EVCONF-CHECKED: ownership candidates found by the analysis, checked at run time; a faster line is being audited) | — | **1.81×** (EVCONF line of 7 fields, compiler-checked, run-time guarded) |
| Chromium | — | 1.39× | not re-measured | | | |

- SQLite (3), (4): leg zg64b. Inputs still named by hand (audit A61f): `BtShared.mutex`, `removeFromSharingList`, the allocator names, `iDb`, `CellInfo`, the mutex API, 12 field names.
- Redis (2): leg rcs4. memcached (1): leg mcnf4 (N1 off: it costs ~5 % here); (2): leg kcj4, root `tsan-ecc-835062685086` (audits A70-A70d; residual L-1'; A/A 1.001); the loop-hoisted line kcn (root `9c656c7b6d62`, audits A70e-A70f, preservation PASS) is in its final gate; (4): leg mct4.
- FFmpeg (2): leg ffb4, 1.29× over same-compiler stock for both the record line and the shipping line (copy_passthrough 2.64×, encoders ~1.00-1.05×); upstream stock lies within 1 % of it.
- One compiler configuration and runtime serves all apps; where it builds without -wp (SQLite, MySQL, FFmpeg) it costs nothing by construction (≤ 284 start-up hook calls), and on Redis and memcached its cost is inside the A/A and −1.6 %.

## Table 2. Optimizations that gain

Rows above EA-CONTENTS: single levers over P1-v3 (the paper's analyses with every soundness fix). Below: on top of their app's configuration.

Kind: S static · R run-time checked, no annotation · G generated from the program's assertions · A hand-written annotation (A+R: run-time guarded) · RT runtime-only · U unsound, ceiling only. Markers: 🟢 gain beyond the A/A (the base run twice) · 🟡 1-2 % or unresolved · ⚪ within ±1 % · 🔴 loss · — not measured. `f` = Intel Xeon w9-3495X, otherwise AMD. Aliases in parentheses.

| optimization | kind | what it is | SQLite | memcached | Redis | MySQL | FFmpeg |
|---|---|---|---|---|---|---|---|
| **DynSTC-RT** | R | single-thread mode in the runtime: nothing recorded while one thread is alive | ⚪ +0.4 | ⚪ −0.3 | `f` 🔴 −2.2 | ⚪ +0.1 | `f` 🟢 **+12.0** |
| DynSTC (compile-time guard); DYNSTC-DIRECT | R | the paper's guard; both forms together | — | ⚪ −0.5 | 🔴 −5.5 | ⚪ −0.7 | `f` 🟢 +10.1; both +31.9 |
| **N1** | S | TSan's "already recorded?" test inlined; the runtime called only on a miss | ⚪ −0.8 | ⚪ +0.3 (🔴 −5 on the record line) | `f` 🟡 +2.5 | 🟡 −1.7 | `f` 🟢 **+5.0** |
| **N1-ST** | R | N1's test skipped in single-thread mode (over N1 + DynSTC-RT) | ⚪ +0.2 | ⚪ −0.2 | `f` 🟡 −1.4 | 🟡 −1.8 | `f` 🟢 **+16.9** |
| **FE-INL** | S | TSan's shadow call stack push and pop inlined | ⚪ −0.7 | ⚪ +0.8 | `f` 🟢 **+4.8** | 🟢 **+3.6** | `f` ⚪ 0.0 |
| **FE-INL + VWIDE-loops** (DE-VERIFIED-WIDE) | R | plus check removal inside loops, verified at run time | ⚪ +0.1 | ⚪ +0.6 | `f` 🟢 **+7.2** | 🟢 **+4.6** | `f` 🟡 +1.8 |
| **FE-SINK** | S | function entry moved to the first point that needs the frame (nothing on top of FE-INL) | 🟡 +1.1 | 🟡 −1.7 | `f` 🟢 **+4.1** | 🟢 **+2.8** | `f` 🟡 −1.2 |
| **N1-L** | S | N1 only in loops with at most 20 checks | ⚪ +0.3 | ⚪ −0.9 | `f` 🔴 −2.5 | ⚪ −0.1 | `f` 🟢 **+3.0** |
| **MEMINTR** (MEMINTR-SRC, one-side) | S | a memcpy from a constant or private source checks only its destination | 🟡 +2.5 | ⚪ +0.9 | `f` 🔴 −2.5 | ⚪ +0.3 | `f` ⚪ 0.0 |
| **EA-CONTENTS** | S | a pointer read from a container no longer makes the container shared | — | 🟢 **+1.4** | — | — | — |
| **SWMR-ROOTS** (SWMR-1) | S | no checks on reads of a global whose only write precedes every reader thread | — | 🟢 **+0.9** | — | — | — |
| **EVCONF** | A+R | objects owned by one event-loop thread unchecked while a run-time guard holds | — | 🟢 **+31.1** with the two rows above | — | — | — |
| **EVCONF-RANGES** | A+R | the guard on memset/memcpy of an owned object | — | 🟢 **+18.5** | — | — | — |
| **EVCONF-ARGS** | A+R | the confinement carried into the hash function's key argument | — | 🟢 **+7.3** | — | — | — |
| **EVCONF-INTERCEPT** | A+R | the guard around memchr, strlen, bcmp on the confined read buffer | — | 🟢 **+7.5** | — | — | — |
| **Quiet threads** (QUIET-THREADS, REDIS-MAIN, AUTO-BIO: silent-thread mode, the Redis phase guard) | R | Redis's main thread skips its checks (ranges included) while every other thread is quiet since a release it acquired | — | — | 🟢 **+31.9** | — | — |
| **LO-OBJ-G**, generated spec (gen6, gen8) | G+R | objects protected by their owner's lock unchecked while the thread holds it; over stock | 🟢 **+15.7** | — | — | — | — |
| **LO-OBJ-G**, spec v7 | A+R | the same, hand-written spec; over stock | 🟢 **+25.0** | — | — | — | — |

## Table 3. No gain

| idea | kind | what it is | result |
|---|---|---|---|
| Removal-mode DE (DE-REMOVAL), DE-3R, DE-2R | S | covered checks deleted; one range check per loop; adjacent fields merged | ⚪ MySQL +0.3 %, SQLite 0, memcached 0, FFmpeg −0.3 %; 🔴 Redis −3.8 % |
| All-paths DE + cycle cut (DE-ALLPATHS, DE-5) | S | covered if every path has a cover; covers kept around loops | ⚪ Redis +1.0 %, SQLite −0.6 %, MySQL +0.1 %, memcached 0 |
| EA-SEND, typed indirect calls | S | `transmit()`'s msghdr writes unchecked | ⚪ memcached 0 (mse4) |
| Loop guard (T8, DE-LC), T10 package | S | a loop-invariant access checked once per sync-free stretch; the older lever package | ⚪ Redis +2.2 / +3.1 %, memcached 0 / +1.6 %, MySQL −0.1 / −4.2 %, FFmpeg +1.4 % |
| Levers on top of SQLite's LO-OBJ-G | S | N1, N1-L, MEMINTR, FE-INL | ⚪ FE-INL +4.5 %, MEMINTR +3.5 % (wide A/A); 🔴 N1 −18.5 % |
| The old "same location" rule (SAME-LOCATION, DSL) | U | fields (S), elements (A), narrow-covers-wide (Z) | 🔴 S loses a memcached race stock reports (`do_item_link` / `clock_handler`); bounds in table 5 |
| LO-OBJ-ARGS; LO-OBJ-RANGES | A+R | LO-OBJ-G into SQLite's record comparison; its guard on memcmp and VDBE memcpy | ⚪ ceiling 1.023; reaches neither site |
| EA interceptor toggle in per-unit builds | S | without the whole-program definitions list | ⚪ SQLite 0.977, MySQL 1.003 |
| memcached's flag globals (SWMR-FIELD) | S | unlocked reads of `settings`, `expanding`, `hashpower` | U ceiling +4.5 %; no sound route; field rule ≈ 0.13 %, dropped 6 Oct (audit A68) |
| Redis I/O threads' client buffers | — | their checks on the buffers they drain | ⚪ ≈ 0: their cycles are a spin |
| Quiet mode beyond Redis | R | the phase guard on the other four apps | ⚪ ≤ 0.4 % of checks in quiet intervals |
| MySQL THD owner reads (THD-READS, X3) | S | a connection's own THD fields | ⚪ ≤ 2.35 % of checks |
| Run-time owner tag for one-thread objects | U | skip the owner's accesses | unsound: no record for a later foreign access |
| N1-ATOMIC | S | inline test for relaxed atomics | ⚪ SQLite, MySQL ≤ 0.65 % of cycles; Redis +1.6 %, unresolved |
| nosync census, DE-5…DE-8, IPA-DE | S | finer cover-breaking rules; callee check covers caller | ⚪ ≤ 2.08 % of checks; −0.8…+1.4 % |
| VWIDE, VWIDE-loops alone | R | run-time verified removal where no check dominates | ⚪ −0.9…+1.3 % |
| FE-hot, hot-list N1 (PGO-PLACE), N1-S, N1-LOOPS-∞ | S | FE-INL or N1 only at hot sites | 🔴 none beats the full version |
| FE-PM, N1-PM, N1b | S | out-of-line entries saving fewer registers | 🔴 MySQL −2 %, SQLite −2…−4 % |
| FE-LAZY, FE-INL-CSE, N1-CSE, LIBCALL-INLINE, N2 | S | lazy entry, shared thread-state load, compact/libc-call/batched checks | ⚪ ±1 %; N2 as built loses races |
| SUBS, SUBS-SEL | RT | a covering record of the same thread counts as a hit | 🔴 −2…−18 %; SUBS-SEL unsound |
| STC-SUM, STC-TS, STC-WL, libfacts | S | closed-world facts for STC and SWMR | ⚪ < 1 % of checks |
| LTO, Attributor, XPASS, XP2 | S | more optimization before instrumentation | ⚪ nothing; the Attributor miscompiles |
| CLONE-ESC, EA-SLOT (EA-SLOT-CONTENTS), ICALL-A2, returns-fresh, field chase | S | finer escape analysis | ⚪ each < 2 % of checks |
| MYSQL-WP, TOPDOWN-FACTS | S | MySQL summaries; cross-unit argument facts | ⚪ ≤ 0.14 % of checks |
| SWMR-G, PUBLISH-ONCE, thread roles, THREAD-IDS | S | finer may-happen-in-parallel facts | ⚪ each < 3 % of checks |
| NOALIAS, CUSTOM-SYNC, MySQL sysvars, INNODB-LATCH, ODR-TRUST | S | language and library facts | ⚪ each < 3.2 % of checks |
| RT-SYNC, SLOT-CHURN (SID-ALIAS), RANGE-OVERWRITE, RANGE-OWN-FASTPATH | RT | runtime-only changes | ⚪ no effect; RANGE-OWN-FASTPATH ≤ 0.3 %, not pursued 6 Oct |
| LO-F, LO-W, LO-B1/B2, sound loop ranges, DE-1, DE-2, DE-4, DE-9 | S | further LO and DE rules | ⚪ ≤ 1.4 % of checks, or unsound |

## Table 4. Open

| item | kind | app | status |
|---|---|---|---|
| **EVCONF-CHECKED**: analysis proposes ownership candidates; run-time owner checks and a shadow marker make each elision sound (a failed check voids the run) | R | memcached | root `tsan-ecc-835062685086`, audits A70-A70d (fit to quote; residual L-1'); screening kcs0 ≈1.36× over stock; leg kcj4 running; get-path ARGS (kcl) next |
| **SQGEN-AUTO**: the generator derives the lock API | G | SQLite | gen8 derives enter/leave and predicates; audit A61f; same code as gen6 |
| **EVCONF-FIELDS**: the owner's reads of its connection's fields | A+R | memcached | +3.6 % on the EVCONF line; needs the wider P-X86-FD |
| QUIET-FE: entry/exit not recorded while Redis's main thread skips | R | Redis | FE 10.3 % of the main thread's cycles; not built |
| DE-AV (SAME-PTR, DE-10): covers checked at run time where no dominance holds | R | SQLite | U ceiling +8-11 %; ≈ +2-4 % expected; parked |
| LO-OBJ-G under `db->mutex` | A+R | SQLite | U ceiling +5.8 %; 0 % reachable in this harness; parked |
| Camera-ready legs | — | all | compiler `tsan-cr-8345a0396fa5` (audits A42-A58b); held for the go |

## Table 5. Where the analyses are conservative (shares of executed checks)

| analysis | what it cannot prove | SQLite | memcached | Redis | MySQL | FFmpeg |
|---|---|---|---|---|---|---|
| DE, dominance | same address twice, no acquire between, both kept | 3.35 % | 0.04 % | 3.44 % | 5.45 % | 5.92 % |
| of which: a call between | callee not proven sync-free (external or indirect / local) | 1.67 / 0.69 % | 0.04 / 0 % | 3.38 / 0.00 % | 2.78 / 1.06 % | 3.53 / 0.57 % |
| of which: the address question | only address equality unproven | 1.0-1.7 % | 0 | 0.06 % | 1.6-2.7 % | 1.8-2.4 % |
| DE, post-dominance | a call breaks it / ceiling with calls allowed | 0.28 / 0.37 % | 0 / 0 % | 0.30 / 0.92 % | 0.02 / 0.53 % | 0.21 / 0.53 % |
| DE, availability | neither check dominates | 4.36 % | 0.08 % | 5.79 % | 1.00 % | 8.56 % |
| DE, all paths / cycle cut | a cover on every path; a cover lost around a loop | 0.22 / 0.92 % | 0 / 0.71 % | 0.82 / 0.14 % | 0.07 / 0.08 % | 0.5 / 0.89 % |
| DE, stronger alias analysis (DE-AA) | must-alias from SCEV or points-to | 0 | 0 | 0 | — | 0 |
| EA, all (EA-TL) | one-thread or consistently ordered memory | 72 % | 55 % | 63 % | 49 % | 70 % |
| EA, pointer parameter | share of the row above | 53-75 % | 53-75 % | 53-75 % | 76 % | 53-75 % |
| EA, heap reachable from shared structures (EA-HEAP) | not provable statically; freed in the allocating call ≤ 0.6 % | 78.7 % | 34.0 % | 41.9 % | — | 7.5 % |
| EA, cross-unit parameter facts | upper bound | 0.04 % | 0.05 % | 0.00 % | 0.14 % | 0.01 % |
| EA, own stack | the thread's own stack | 4.9 % | 5.9 % | 12.5 % | 14.8-24 % | 5.8 % |

The old "same location" rule (U; removal mode, over exact DE):

| variant | Redis (`f`) | SQLite | memcached (`f`) | MySQL | FFmpeg, checks removed |
|---|---|---|---|---|---|
| S: fields of one struct | +8.4 % | +11.9 % | +1.7 % | +4.1 % | 13.7 % |
| A: elements of one array | +5.9 % | +12.8 % | +3.9 % | +1.1 % | 22.1 % |
| S + A | +11.5 % | +27.3 % | +6.6 % | +6.0 % | 34.4 % |
| S + A + Z (narrow covers wide) | +10.6 % | +31.2 % | +6.8 % | +7.9 % | 35.4 % |

## Table 6. Every other optimization tried

| optimization (aliases) | kind | what it is | result | standing |
|---|---|---|---|---|
| **DE** | | | | |
| DE package (T4, T4+E), DE-B selective peeling | S | merge + loop ranges + selective peeling | with exact DE: memcached `f` +3.6 %, MySQL +3…+6 %, Redis −2.6 %, others ±1 % | superseded |
| N3, N4, N8 | S | DE path flags; covers across iterations; trimmed entry/exit | ≤ 1 % each | closed |
| DE-8b | U | C allocation functions as sync-free | 0 % | dropped |
| shadow-proxy rule (RedCard) | U | a cover reports a different race than stock | 0.1-1.1 % of checks | not adopted |
| relaxed-atomic rule (`-tsan-de-relaxed-atomic-nosync`) | S | relaxed atomics do not break a cover (A8) | SQLite loop-guard skips 0.74 → ≈ 9.2 % | on in every build |
| pure-asm rule; anticipation, acquire tolerance, available-checks dataflow | S | further cover placements | ≈ 0; ≤ 0.56 % | closed |
| POSTDOM-TERM | S | post-dominance with proven termination | ≤ 0.27 % | in the tree |
| **EA** | | | | |
| U2 | U | named arguments never escape | 0.963-0.996 over T1 | closed |
| integer-copy closure off (A7) | U | pointers copied as integers not followed | memcached +0.9, Redis +1.0, MySQL −2.7 % | closure kept on |
| EA-7; record-compare chain | U | arguments of external or address-taken functions | ≈ 5.6 % of SQLite's checks | shelved |
| EA-WP | S | whole-program summaries | 0 % of the heap mass | closed |
| EA call-site flags (MAAP-style), OWN-STACK-ENTRY (X2) | S | per-call-site pointer-parameter facts | MySQL 0.01 %, bound 3.01 %; entry ceiling 24.4 %, unsound | parked 5 Oct |
| **STC, SWMR, LO** | | | | |
| STC-1…STC-4, join-aware STC, STC-CALLEE, STC-STRONG (i) (O-STC), DYN-1 | S | finer STC rules | ≤ 2.44 %, mostly ≈ 0 | closed |
| MAIN-ONLY (STC-STRONG (ii)) | S | Redis's reads only on the main thread, statically | impossible statically; taken at run time by the quiet threads | closed |
| DYN-2 | R | where DynSTC's guard skips | only FFmpeg's stream copy | closed |
| LO-unwind; LO-C | S | unwind edges; POSIX calls transparent to LO | refuted; no gain | closed |
| SQLITE-CW | S | closed world for SQLite | 0.02 % | closed |
| SWMR-H | S | written only before publication | ceiling ≤ +0.2 %; 14 lost-race paths | parked |
| SWMR-ROOTS on SQLite, MySQL, Redis | S | memcached's rule elsewhere | ≈ 0-1.3 % reachable | closed 5 Oct |
| G-EA | A+R | EA under LO-OBJ-G's guard | −3.7 % | closed |
| LO-OBJ-G spec completion; SQLITE-KEY | A+R | unnamed BtShared fields; comparator key and payload | ≈ 1.2 % / ≈ 3 % of checks | not built; parked |
| LO-OBJ-G Pager/Wal roots | A+R | Pager and Wal under the b-tree's mutex | 12.7 % of the locked mass | declined (P-PAGER) |
| SQLite sharable guard; PERELEM | R; S | per-object shared-cache guard; per-element locksets | cannot reach the mass; provable ≈ 0 | closed; parked |
| TLS-rooted escape analysis (LO-TID) | S | objects reachable only from a `thread_local` root | MySQL ≈ 0.8 % | parked |
| **N1, FE and other inline paths** | | | | |
| N1-ST =miss; N1-ST (b), (c) =fs | R | flag read on a miss; hoisted; in the fast state | FFmpeg −6.6 %; equal | front kept |
| N1-ST-WORKER; N1-SPLIT, N1-PAIR; N1-CSE across calls | S | inline-test variants | 0; subsumed by DE-2R; ≤ 1 % expected | closed; not built |
| FE-INL exit-max=1; unified exit | S | FE-INL variants | MySQL −5.2 %; identical | dropped |
| FE oracle; TSan-aware inlining | U | entry/exit removed | Redis +23.5 pp, MySQL +13.6, SQLite +4.4 | ceiling |
| call-cost ceiling (N1/N2-real) | U | time in runtime calls of hitting checks | SQLite 21 %, FFmpeg 17.5 %, Redis 25 %, memcached ≈ 0 | bounds N1, N2 |
| FE-NOFRAME | S | no entry/exit without accesses | — | not taken: changes report stacks |
| MEMINTR-INLINE | S | small memcpy/memset inlined | ≤ 1 % | not recommended |
| **Ownership variants** | | | | |
| EVCONF-INTERCEPT write side (X5) | A+R | the guard on sendmsg/writev iovecs | ceiling +0.7 % | closed |
| fresh item until linked; `_nosrc` copies | A+R | memcached's new item; covered-source copies | ≤ 1.1 %; ≤ 0.8 % | not built |
| quiet-thread refinements | R | skip bit in the fast state; re-read after sync calls | ≤ 0.3 % | parked |
| R1-R7 | — | full-connection claim, allocation stacks, stats mutex, SQLite latch, refcount readers, DOM-FS, guard hoist | unsound, report-changing, or ≤ 0.3 % | rejected 4 Oct |
| OWN-HANDOFF | A | skip while the program's hand-off protocol says one thread holds a buffer | Redis +13.9 % | replaced by the quiet threads |
| OWN-CONN | A+R | owner guard on SQLite's private connections | ≈ +21 % estimated | withdrawn (P-CONN) |
| OWN-STACK; SPIN-ACQ | R; RT | own-stack skip; spin acquires only on a change | ≈ −1 %; Redis ≤ +14 % ceiling | parked |
| FFmpeg heap hand-off | A | buffers handed between threads | program-protocol premise | closed |
| **Runtime-only track** (parked from the paper) | | | | |
| DD-EXACT (DD-COST; DD-CHEAP, DD-BIG), DD-STATIC | RT | TSan's deadlock detector | DD-CHEAP null; DD-BIG memcached +21.7 % but drops lock-order reports; DD-STATIC out of scope | parked |
| RT-SYNC-LF | RT | lock-free acquire loads | unresolved | parked |
| RT-RANGE, RANGE-HIT-SKIP, RANGE-VEC, RANGE-UNIFORM | RT | faster range checks | ≤ 0.5-2 % | closed or parked |
| RT-ALLOC | RT | faster allocation bookkeeping | Redis ≈ 11 % of cycles (mass) | parked |
| RT-SLOT, SLOT-PREF, empty-epoch elision, early reset | RT | slot preemption, global resets | ≤ 1.7 % | closed |
| ALLOC-COVER, RANGE-COVER | RT | allocation and range records as covers | 5.6-7.0 % (proxy) | out of scope |
| sendmsg batching, interceptor bypass, EA-P1's out-parameter cost, EA precision counts | — | — | no result yet | — |

## History: superseded results

| item | was | now | why |
|---|---|---|---|
| Redis quiet threads, hand-written lines | 1.39-1.40× | 1.54× derived (AUTO-BIO) | lines derived from the whole program, 5 Oct |
| Redis quiet threads, local rule only (AUTO) | 0.946-0.962 of the best | AUTO-BIO | AUTO cannot prove the background thread's start |
| Redis, derived lines on the earlier root | 1.45× | 1.54× (rcs4) | the one configuration, 6 Oct |
| SQLite spec v5, v6 | +18.8 %, +23.5 % | v7 1.25× (zg64b) | v7 carries the latch; audited |
| SQLite generated spec gen5 | 1.14× (zg5st4) | gen6/gen8 1.16× | gen6 adds 64 nocopy lines |
| memcached EVCONF line (cwn) | +80.3 % | 1.81× (mct4) | the one configuration, 6 Oct |
| memcached without annotations | 1.01× | 1.02× (mcnf4) | + FE-INL, MEMINTR; N1 off |
| memcached spec derived from the program (EVCONF-DERIVE, zma) | 1.010 over the hand spec (zma4) | parked 7 Oct; EVCONF-CHECKED | audit A59e: unsound; the fixed derivation derives 0 fields |
| memcached one configuration with the inline hit test (mcq) | 0.922 over zmr | without it, −1.6 % | N1 costs ~5 % on memcached |
| FFmpeg, MySQL, SQLite one configuration with -wp | 0.991 (ffr4); 0.994 / 0.987 (myr4); unresolvable (sqxy4) | no -wp | -wp rule, 7 Oct |
| Removal-mode DE on FFmpeg | +10.2 % (screening) | −0.3 % (full leg) | did not reproduce |
| Compile time, per unit | SQLite +20 %, memcached +8 %, Redis +9 %, FFmpeg +16 % | table below | omitted the -wp summary step |
| Compile time, first rows on 744024407b56 | Redis +324 %, memcached +121 % | table below | method fixed 7 Oct |

## Compile time

Root `tsan-cc-744024407b56`, both arms from the same tree; -wp summary step + build, no ccache, medians of 3. Rows: `$EXTRA/wt-dev2-r/cct/btime.txt` (`$EXTRA` = the lab host's `/extra/<user>`).

| app | line of record (tree) | -j | stock | line | overhead | of which summary step |
|---|---|---|---|---|---|---|
| memcached | no annotations (mcf) | 8 | 6.07 s | 11.01 s | +81 % | 4.67 s |
| memcached | one configuration + EVCONF (mcy) | 8 | 6.16 s | 14.83 s | +141 % | 8.53 s |
| Redis | one configuration (rcy) | 8 | 5.06 s | 22.91 s | +353 % | 13.06 s |
| SQLite | LO-OBJ-G gen6 (zg6) | 8 | 32.88 s | 41.63 s | +27 % | (no -wp) |
| FFmpeg | ffn | 8 | 77.30 s | 139.29 s | +80 % | (no -wp) |
| MySQL | Release, FE-INL (rpf) | 6 | 858.30 s | 1005.21 s | +17 % | (no -wp) |

- FFmpeg: N1's inline hit tests give 3.8× stock's .text; on 40 units the analyses add 17 % compile CPU, the hit test ×1.96. MySQL: FE-INL grows .text ×1.86.
- Fixed: the phase-summary pass on FFmpeg 249.8 → 37.5 s (510cbba6ec58, identical records).
- **Lock-scope memo (P0, a0c3b51bbe07, root `tsan-cc-a0c3b51bbe07`; gate passed 7 Oct).** The phase derivation now computes the set of functions that may release a mutex once per round, not once per function. Old root vs new root, the same tree, alternated in each rep, CPUs 12-19, -j8, medians of 3:

  | line | old root | P0 root | summary step |
  |---|---|---|---|
  | memcached mcf | 12.70 s | 12.62 s | (no phase pass) |
  | memcached mcy | 15.58 s | 13.02 s | 8.98 → 6.04 s |
  | Redis rcy | 24.66 s | 21.50 s | 13.98 → 11.01 s |

  - **Identity:** 12 pairs, covering memcached mcf and mcy and Redis rcy × 3 reps, plus SQLite zg6, FFmpeg ffn and MySQL rpf × 1.
    - Every linked binary has the same code bytes and allocated section sizes.
    - Redis's one exception is by design: the phase guard's program-hash constant at 90 constructor sites. The program digest folds in each unit's recorded compile line, which names the compiler.
    - The summaries are the same, apart from the digest and the phase record that holds it. Given the old build's own IR, the new root re-derives every summary file byte for byte, the digest included.
  - Rows, gate and scripts: `$EXTRA/de-recovery-gate/degen/ctime/p0/` (btime.txt, gate.txt, ctp0.sh, elfcmp.py, sumcmp.py, rederive.sh).
  - These rows are a separate run from the table above, so compare within this table only.
- Parked (7 Oct): configure once for memcached (P1a, its gate written, not run). Per-module summaries and compile-once are deferred. PGO-PLACE for FFmpeg is in progress.

## Notes

- Race preservation: 10 stock vs 10 optimized runs per app, no race-report site lost (memcached 4 kept, SQLite 3, MySQL 232-236, Redis all seeded; FFmpeg's stock reports none).
- memcached's workload V4 was chosen among five (2 Oct): default +2.8 %, pipelining +3.5 %, 32-key gets +6.3 %, long keys +16.8 %, the mix +30.4 %.
- Premises awaiting a ruling used by results of record: P-X86-FD (narrow), P-LIBEVENT, A5, A8, A9.
