# TSan instrumentation optimizations: results

State: 7 Oct 2026. Longer earlier versions: 90a735e, f02509d, b8d5c56. Premises: `soundness-fixes.md` §6.

## Table 1. Summary (AMD, 2 × EPYC 9115)

Speedup = stock TSan's run time over the optimized build's, same leg, geometric mean over 4 code offsets. The stock
build is by the same compiler and runtime unless the cell says "over upstream" (upstream = LLVM at the fork point
plus two upstream ThreadSanitizer.cpp commits, `tsan-pristine-up59b2-0f1aed148576`). Our stock and upstream stock
differ by -4 to +1.5 % across the apps (measured separately). Every figure is the best sound one measured (no race stock TSan reports is lost,
under the premises of §6).

| app | workload | submitted paper | (1) no annotations, static | (2) no annotations, run-time checked | (3) spec generated from the program's assertions | (4) our annotations |
|---|---|---|---|---|---|---|
| SQLite | threadtest3: stress2 | 1.71× | ≈ 1.0× (the paper's analyses; 0.98× over same-compiler stock, inside the A/A) | **1.14× (AMD) / 1.06× (Intel) measured, not yet a claim** (LO-MARK v1: lock markers in shadow, no assertions and no annotations; see the note) | **1.19×** (LO-OBJ-G, spec gen9b = sqgen16's output; run-time lock guard); **1.26-1.29× preliminary** with the parser parameters derived and bounds checked at run time (gen19f / gen19j; see the note) | **1.25×** (LO-OBJ-G, spec v7; run-time guarded) |
| FFmpeg | four transcodes of one film | 1.57× | — | **1.29×** (DynSTC-RT + N1 + N1-ST) | — | — |
| Redis | 7 data-heavy commands, 8 I/O threads | 1.45× | 1.10× (FE-INL + N1) | **1.54×** (FE-INL + N1 + quiet threads derived from the whole program) | — | 1.40× over upstream (hand-written quiet-thread lines; superseded by (2)) |
| MySQL | Release, sysbench insert / update / delete, 24 connections | 1.16× | **1.13×** over upstream (FE-INL) | — | — | — |
| memcached | pipelined 32-key gets, 190-byte keys (V4) | 1.07× | **1.02×** (the analyses + EA-CONTENTS + SWMR-ROOTS + FE-INL + MEMINTR) | **1.80×** over upstream (EVCONF-CHECKED: ownership candidates found by the analysis, checked at run time; repaired runtime, leg kmg6) | — | **1.81×** (EVCONF line of 7 fields, compiler-checked, run-time guarded) |
| Chromium | — | 1.39× | not re-measured | | | |

- SQLite (3), preliminary 1.29×: leg zcf4 (8 Oct, root tsan-carve-ad1ae6e3aa43, 4 offsets × N=4, A/A 1.012). Over same-root stock: gen9b 1.255×, gen19f 1.292× (1.226-1.346); gen19f over gen9b 1.030 (1.017-1.042, above the A/A at 3 of 4 offsets). gen19f derives the three parser parameters without hand lines and replaces the sub-array premise by a run-time carve check (39.6-53.3 thousand carvings per run, no void; its cost is not measurable). Preliminary because that binary narrows the carve check per parameter, which audit A73 found unsound in general; the confirm leg runs the final form (root tsan-carve-b575340c1ee8: carve checked whole, member-array bounds tested). gen9b reads higher on this compiler line than on 8345a (1.255 against 1.19) because stock is slower there (leg zrc4).
- SQLite (3), confirm leg zdc4 (8-9 Oct, the final check form: root tsan-carve-b575340c1ee8, carve checked whole, member-array bounds tested): over same-root stock gen9b 1.200×, gen19j 1.260×, gen19j with bound tests on the cursor's two arrays as well 1.230×; gen19j over gen9b 1.050. Another user's load on the leg's half retired 29 of 80 cells; the clean cells cover every arm at all four offsets, but at N = 2-3 the A/A is 0.986 with one offset at 0.906, so this leg confirms the direction, not the digit. The cursor-array bound tests read 0.976 of gen19j on the mean (inside that wide A/A): no resolved cost, no proof of none; the cursor arrays therefore stay a named premise for now. A quiet re-run decides the figure of record. Every run logged carvings and no void.
- SQLite (3) on the Intel host (9 Oct, leg zdf4, same root, clean: 0 of 80 cells retired, 4 rounds x 4 offsets): over stock gen9b 1.093×, gen19j 1.109×, gen19j with the cursor-array bound tests 1.131×; A/A 1.003 with a per-offset range of 0.93-1.10. The three specs are not separated there, and the cursor-array tests read 1.020 of gen19j against 0.976 on the AMD leg: their cost is not resolved on either machine. Gains on Intel are about half the AMD ones for every SQLite spec.
- SQLite (3), (4): leg zgb4 (4 offsets × N=6, A/A 1.005; gen6, the earlier (3), reads 1.17× in the same leg). (3) names no SQLite struct, field or function except `BtShared.mutex`; the generator also names the sqlite3_mutex_* API, malloc/free and __assert_fail (audits A61h-A61m; premise PM, §6). (3) leaves ~5 % against the hand-written spec (0.951); three parser parameters written by hand recover ~3/4 of it (diagnostic leg zgp4: 1.048× over (3)), but deriving them needs at least three further generator rules and one premise, so they are not in (3). Preservation for the parser-parameter spec (gen19f with the carve check, root tsan-carve-ad1ae6e3aa43; 8 Oct): the full 15-test suite, 10 stock against 10 runs, no site lost (4 sites at the finest level, all kept; three are WAL-index races that stock itself reports in 1-7 of 10 runs); 64-219 thousand carvings per run, no void. create_drop_index_1 is not quoted: its A/A reads 1.18× (the same binary varies 0.92-1.36×). Measured on half A (CPUs 8-15,40-47: socket 0, local to the NVMe, 16 hardware threads). Roots: the same flags read 4-5 % lower in absolute speed on the e52d compiler line than on 8345a, for stock and line alike (leg zrc4: stock 0.934, line 0.955, A/A 0.998; compiled code identical), while the ratio over same-root stock holds (1.20 and 1.23). The cause is open. The compiled code is identical, and the runtime's access entries are no longer on e52d. One measurement on half A pointed at code placement (8345a's stock with its runtime padded by e52d's extra 26,080 bytes of text, so every program symbol sits at e52d's address, ran like e52d's stock: 0.903 against 0.899; 8 + 8 + 4 runs, 8 Oct). A second run on half B did not reproduce it (the same padded binary read 1.014 of the unpadded one; six page-scale pads, 4 runs each, run-to-run noise 8 %, an identical pair 4.6 % apart). The two are not reconciled; it needs 10 or more runs per pad on a quiet half A with e52d's stock in the same rotation. Until then the gap is not attributed, and SQLite legs keep the four byte-level offsets. On half B (socket 1, 32 threads) stock's stress2 runs ~3.5 % faster while gen6's does not, so gen6 reads 1.10-1.12× there instead of 1.17×.
- Redis (2): leg rcs4. memcached (1): leg mcnf4 (N1 off: it costs ~5 % here); (2) **was repaired on 8 Oct and re-timed on 9 Oct** (leg kmg6, root `tsan-ecc-b54ce0196884`, 4 offsets, two on each half: 1.798× over upstream, per offset 1.825 / 1.814 / 1.817 / 1.737; A/A 1.001; the fourth offset's lower value is that binary's placement, reproduced by its A/A arm; 0 voids in 32 runs; preservation PASS). What was repaired: a code reading found that the run-time's lazy clear of a released owner marker lets any ordered access replace the marker, also a read or a write narrower than the granule; a later access unordered with the owner's elided write can then pass where stock would report (audit A77b D1, confirmed in the EVCONF-CHECKED code; no test covered the shape, and preservation saw no lost site in the measured runs). The fix keeps the marker unless a whole-granule write replaces it; a second hole of the same family (a new key marking over another key's released marker without an ordering test, reproduced on both timed runtimes) is fixed with it. Screening leg kmg5 on the first fix: 1.815× (1.799-1.825) against 1.806× for the old runtime, fixed/old 1.005, A/A 1.001; in memcached every one of the 46 million lazy clears observed at paper scale is a whole-granule write, so the repaired rule changes nothing there. The figure of record will come from a frozen one-commit root (tsan-ecc-b54ce0196884, leg kmg6). Until then: leg kmg, 1.801× (1.792-1.812) over upstream stock, root `tsan-ecc-9c656c7b6d62` with the marker gate (`experiment/evconf-markergate` 3a37f1be3df6; audit A70g; gate record `evconf-checked.md` §14: check-tsan ×12, preservation PASS). Without the gate the same line reads 1.757× (leg kcn record, reproduced in kmg); the gate moves EVCONF's marker handling out of the shadow-check fast path, which had made plain stock on that root ~10 % slower (now ~4 %). Over stock of its own root the line reads 1.96×, which counts that regression and is not quoted. Residual L-1'; the earlier root without the loop-hoisted check read 1.34× (kcj4); (4): leg mct4.
- FFmpeg (2): the record line ffk reads 1.285× over same-compiler stock (root 744, leg fft4b; ffb4 1.283×), copy_passthrough 2.68×, encoders ~1.00×; the shipping-compiler line reads 1.290× over stock of its own root; upstream stock lies within 1 % of ours. Of the 1.29×, 1.085× is DynSTC-RT's runtime option alone (stock TSan run with dynstc_rt=1 gains the same; leg fft4b), and the instrumentation 1.18×. FFmpeg's record workload reports no race in stock, so preservation rests on seeded races (8 Oct, `ffmpeg-seeded-design.md`): 8 seeds, one per mechanism the line changes (DynSTC-RT transitions, N1 inline test, N1-ST skip, copy_passthrough's hot path, DE-covered accesses), each reported 10/10 by stock and by ffk and s1; seeds off, 0 reports.
- One compiler configuration and runtime serves all apps; where it builds without -wp (SQLite, MySQL, FFmpeg) it costs nothing by construction (a few hundred start-up hook calls at most in short probes, none after the first thread), and on Redis and memcached its cost is inside the A/A and −1.6 %.

**SQLite column (2), LO-MARK v1, first timing (9 Oct, leg zvf4, Intel host, stress2, 4 offsets x 4 rounds, same root `tsan-lomark-718c7520052c`).** LO-MARK over stock 1.058 (per offset 1.047 / 1.045 / 1.056 / 1.082); gen9b over stock 1.095 on the same root; LO-MARK over gen9b 0.966; A/A 0.986. Another user's job disturbed 38 of 64 cells; on the clean cells alone: 1.054, 1.100, 0.981, A/A 0.988 (LO-MARK then has three offsets at N <= 2). Second leg (zvl6b, AMD host, the noisier half, 6 rounds, 121 cells, none void): LO-MARK over stock 1.140 (1.214 / 1.145 / 1.104 / 1.101); gen9b over stock 1.174; LO-MARK over gen9b 0.971; A/A 0.993. On both machines LO-MARK v1 is about 3 % below the generated spec at every offset and well above stock. All 16 + 24 LO-MARK runs: 0 voids; one run has 10.7 M held hits, 10.7 k marks, 317 k accesses stored beside a marker. v1 covers MemPage, the page image, the temp space and BtShared (about 61 % of what gen9b elides); cursors are v2, built and audited (A80, A81), not timed. The figure is a measurement: the six premises of v1 await a decision, and the record leg on the quiet AMD half is not done.

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
| **EVCONF-CHECKED**: analysis proposes ownership candidates; run-time owner checks and a shadow marker make each elision sound (a failed check voids the run) | R | memcached | root `tsan-ecc-9c656c7b6d62` + marker gate, audits A70-A70g; **1.80×** over upstream (kmg; 1.76× without the gate); the one check per walked loop (A70e/A70f) took it from 1.34× |
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
| SQLite path B: lock-held elision without assertions, unheld accesses recorded and checked at run time (candidates by static coverage, C-COVER) | R | BtShared's lock; root `tsan-lob-fbe442ec2e2c`, audit A71 | the coverage rule keeps 4 BtShared fields: 0.41 % of executed checks elided against 21 % for the generated spec; unsound ceiling for 34 fields 1.031× (leg zgf4, A/A 0.995). The gain is in the page bytes, which static coverage cannot reach | stopped 8 Oct; superseded by the lock-marker design (LO-MARK, in progress) |
| N1 placement for compile time: profile hot list (PGO-PLACE q=0.99, 0.999); static frequency (N1-S s1, s4, K=2) | S | the inline hit test only at profile-hot or statically hot checks (block frequency per function entry ≥ 1, ≥ 4; top 2 per function), to cut FFmpeg's compile time | FFmpeg compile overhead +97 % → +46 / +36 / +20 % (static, final table); speed over stock 1.285× → 1.241× / 1.227× (fft4b), k2 ≈ 1.09× (screening); profile 0.963 / 0.989 of full inline. All of the loss is in copy_passthrough | closed |
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
| SQLite zgB with its spec generator (8 Oct) | sqgen16d 529.7 s; line 576.7 s, 17.3× stock (gen0.sh, g0time.txt) | sqgen16e 49.3 s; 96.2 s, 2.89× | sqgen16e, byte-identical output |
| Compile time, first rows on 744024407b56 | Redis +324 %, memcached +121 % | table below | method fixed 7 Oct |
| Compile time on 744024407b56, before P0, P1a, P1c (cct/btime.txt) | mcf +81 %, mcy +141 %, Redis +353 %, SQLite +27 %, FFmpeg +80 %, MySQL +17 % | mcf +11 %, mcy +35 %, Redis +231 %, SQLite +28 %, FFmpeg +97 %, MySQL +16 % (final table) | P0 in the compiler; P1a and P1c in the harness, all identity-gated |

## Compile time

**Final table (7 Oct).** Root `tsan-n1s-e52d8787b59e`: 744024407b56 plus P0 (a0c3b51bbe07) plus the two N1-S commits, which only the FFmpeg variants s1, s4 and k2 switch on. Its runtime is byte-identical to `tsan-cc-a0c3b51bbe07`'s. Freeze gates: IR 196 passed, tsan 410 passed.

Method:
- The harness carries P1a (memcached configures once) and P1c (the summary step links bitcode).
- Stock is built from the same tree in the same run, alternating with the line in each rep.
- Compile = -wp summary step + build, timed by build.sh's ns timer. Fresh trees, no ccache, CPUs 28-35, medians of 3.
- Overhead = median of the line / median of stock.
- Rows: `$EXTRA/wt-dev2-r/cctf/btime.txt`; driver cctf.sh (`$EXTRA` = the lab host's `/extra/<user>`).

| app | line of record (tree) | -j | stock | line | overhead | of which summary step |
|---|---|---|---|---|---|---|
| memcached | no annotations (mcf) | 8 | 7.25 s | 8.03 s | +11 % | 6.74 s (configure included, P1a) |
| memcached | one configuration + EVCONF (mcy) | 8 | 6.47 s | 8.71 s | +35 % | 7.38 s (configure included, P1a) |
| Redis | one configuration (rcy) | 8 | 5.53 s | 18.31 s | +231 % (with P1d: +212 %, CPU +158 %) | 8.23 s |
| SQLite | LO-OBJ-G gen6 (zg6) | 8 | 33.23 s | 42.57 s | +28 % | (no -wp) |
| SQLite | LO-OBJ-G gen9b (zgB), build only | 8 | 33.27 s (CPU 34.1) | 43.28 s (CPU 43.1) | +30 % | (no -wp) |
| SQLite | zgB with its spec generator (sqgen16e), no profile (once per program version) | — | 33.27 s | 96.2 s (CPU ~98.6) | 2.89× | generator 52.9 s |
| FFmpeg | ffn = ffk, full inline hit test | 8 | 76.10 s | 149.88 s | +97 % | (no -wp) |
| MySQL | Release, FE-INL (rpf) | 6 | 874.75 s | 1015.13 s | +16 % | (no -wp) |

**SQLite zgB rows (8 Oct, separate runs; same root and harness).** Stock and zgB are built from zgB__ in the same
run, 3 reps, medians. SQLite compiles one unit, so its CPU is close to its wall. The rows and generator times are in
`$EXTRA/wt-dev2-r/cczgb/` (btime.txt; generator g0time.txt; drivers cczgb.sh, cczgb23.sh, gen0.sh).

- **The generator, per run, with no profile (wall; CPU), 3 reps** (gen1.sh, the record run for column 3's provenance;
  `g1time.txt`):
  - **IR:** sqlite3.c to IR at -O0 twice, a dbg build with -DSQLITE_DEBUG (assertions live) and a rel build, each
    with mem2reg, in parallel: 3.6 s; ~6.3 CPU-s.
  - **sqgen16e.py** (9e08d7dd499d1de1), which writes the spec, with an empty executed-function list: 49.3 s on one
    thread.
  - Total 52.9 s.
  - **Every rep's spec has fe95d761's directives exactly**, and its body equals sqgen16d's from the same IR byte for
    byte. The files differ only in the header, which names that rep's input paths.
  - **The line's own build needs none of it.** The two -O0 IR builds are the generator's alone, including the dbg
    build, which no benchmark build uses.
  - It runs once per program version: the spec is a file, reused by every later build.
  - **Not needed: the coverage step** (a native coverage build and a 120 s workload run, 137.7 s). The function list
    only chooses latch against exclude, and on SQLite it does not change the spec.
- **Identity of the build:** the line built with a generated spec has the same code as one built with
  `sqlite3-gen9b.spec`, 6/6 objects, compared in trees whose names have the same length. The host tools jimsh,
  lemon and mkkeywordhash embed absolute source paths, so a shorter tree name shifts their `.rodata`; the first
  control differed for that reason.

FFmpeg variants: inline hit tests at statically chosen sites (N1-S), same run and stock. Speed is over stock TSan as shipped (no runtime options), from leg fft4b (AMD, 4 offsets × N=1, balanced order; per-offset range in brackets).

| variant | flags beyond ffn | line | compile overhead | (paired per-rep median) | speed over stock |
|---|---|---|---|---|---|
| ffk (ffn) | — | 149.88 s | +97 % | +88 % | **1.285×** (1.266-1.305) |
| s1 (ffu) | static-hot=1e6 min-freq=1 max-insts=0 | 110.77 s | +46 % | +44 % | **1.241×** (1.237-1.246) |
| s4 (ffv) | min-freq=4 | 103.53 s | +36 % | +33 % | 1.227× (1.220-1.234) |
| k2 (ffw) | static-hot=2 | 91.09 s | +20 % | +15 % | ≈ 1.09× (screening: 0.845 of ffk) |

- All of the speed difference is in copy_passthrough (ffk 2.68×, s1 2.34×, s4 2.27× over stock); the encoders are at 0.98-1.02.
- Of ffk's 1.285×, 1.085× is the runtime option dynstc_rt=1, which speeds stock TSan by the same 8.5 % when stock runs with it; the instrumentation line itself gives 1.184× (leg fft4, 1.190×). A/A on the same half in fft4: 0.994.
- FFmpeg's rep 2 ran at load ~10, and its stock build took 94.4 s against 75.1 and 76.1 s; the paired column divides each rep's line by its own stock.
- Overlaps:
  - The first 18 rows (rep 1, and rep 2 through Redis st) overlapped this lane's own SQLite preservation run on CPUs 52-59 (gpresb).
  - f1ffk2j8r2 overlapped a 4-minute offset relink on 52-59 (ffsoff2).
  - Both are logged in each row's `others=`.
- **Static placement remains a negative for speed** (Table 6). It trades compile time against speed; s1 halves FFmpeg's compile overhead (+97 % → +46 %) for 1.285× → 1.241× speed over stock, and the choice among s1, s4 and k2 is a trade-off, not a fix.

- **Where Redis rcy's time goes (8 Oct, measured on the final table's setup; `$EXTRA/wt-dev2-r/rct/RESULTS.txt`).**
  - The cost driver is the inline fast paths, mostly in instruction selection. Over the 89 units the build takes
    +34.0 CPU-s over stock (20.4 → 54.5); the inline fast paths account for 30.5 of them (instruction selection
    19.4 CPU-s across all units). The analyses add 5.9 and the phase flags 3.3; the variants overlap.
  - The TSan pass itself costs 1.75 CPU-s, 0.91 of it in module.c.
  - The summary step takes 8.1 s wall: make -n 1.9, IR emission 1.6, link 1.0, deps 0.8, opt 2.0 (parse 0.75,
    EA 0.76, phase summary 0.33).
  - The build's wall time is also set by order: module.c, the longest unit (5.9 s), starts 58th of 89 in make's order.
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
- **Configure once for memcached (P1a, 7 Oct).** The gate passed: 6 pairs with identical linked code, summaries and configure outputs. Base flow against P1a flow on the P0 root, CPUs 28-35, -j8, medians of 3:
  - mcf 11.44 → 8.06 s (−29.6 %);
  - mcy 12.53 → 9.08 s (−27.5 %).
  - Rows: `$EXTRA/de-recovery-gate/degen/ctime/p1a/btime.txt`. These rows are a separate run, with no stock row.
  - Per-module summaries and compile-once remain deferred.
- **Bitcode link in the summary step (P1c, 7 Oct).** The modules' IR is still emitted as text, so the recorded compile flags and the program digest do not change. llvm-as assembles each module in parallel, then llvm-link and opt read bitcode, which avoids re-parsing about 45 MB of text for Redis. Gate passed: 9 pairs (Redis rcy, memcached mcy and mcf, 3 reps each) with identical linked code and summaries, the digest included.
  - One exception, by design: ST's `# tsan-summary-link:` comment names the file opt read, `.ll` before and `.bc` after. Readers skip it, and the gate masks only that extension.
  - Base flow against P1c flow on the P0 root, CPUs 28-35, -j8, medians of 3, summary step / compile:
    - Redis rcy: 10.62 → 8.00 s / 20.59 → 18.01 s (−12.5 %);
    - mcy: 5.71 → 5.28 s / 12.15 → 11.80 s (−2.9 %);
    - mcf: 4.84 → 4.57 s / 11.28 → 11.02 s (−2.3 %).
  - Base and P1c alternated per rep. Another user's builds ran on the host during the run (load ~6).
  - Rows, gate, patch: `$EXTRA/de-recovery-gate/degen/ctime/p1c/` (btime.txt, gate.txt, p1c-apply.py, ctp1c.sh, sumcmp.py).
  - SQLite's generator still links text, so the same change applies there; FFmpeg and MySQL already link bitcode.
- **Redis scheduling (P1d, 8 Oct).** The gate passed: 6 pairs (Redis stock and rcy × 3) with identical linked code;
  for rcy the summaries and the summary step's compile lines are identical too. Two changes, both in the harness:
  - **(a) No Makefile.dep remake in the summary step.** The step's `make -n` finds an empty `src/Makefile.dep` and
    no longer remakes it. GNU make remakes included makefiles even under -n, by a one-thread `clang -MM` over every
    source. Summary step 7.79 → 6.11 s.
  - **(b) The ten largest objects first, as make goals.** Stock gets the same order in the same run. This gains
    nothing in the harness (build 9.83 → 9.66 s): module.c alone takes about 6.6 s, the whole object phase, so it is
    the critical path whatever the order. The 13.0 → 10.4 s seen in a standalone build from distclean did not
    reproduce here. Keeping (b) is optional.
  - **Rows (root e52d, harness P1a + P1c + P1d, CPUs 28-35, -j8, medians of 3; stock → rcy, both arms ordered):**
    - wall 5.05 → 15.75 s (+212 %), CPU 44.5 → 115.0 s (+158 %);
    - the same run without P1d: 5.06 → 17.58 s (+247 %), CPU 44.2 → 116.9 s;
    - CPU is the children's user+sys of the whole legbuild call.
  - Rows and gate: `$EXTRA/de-recovery-gate/degen/ctime/p1d/` (btime.txt, gate.txt, p1d-apply.py, ctp1d.sh).
  - **What is left on the wall: module.c.** Its TSan pass takes 0.91 s (every other unit ≤ 0.08 s); an
    identity-preserving fix there comes straight off the critical path. Its instruction selection (2.1 s) is the
    inline fast paths' own cost.
  - **Parked lever (8 Oct, not built): memoise classifySyncEffect in DE's scanPaths.** Each covering pair rescans
    the same calls, 8.2 % of module.c's compile. Identical by construction (a pure function of the call, its callee
    and read-only analyses). Expected gain about −0.4 s wall on rcy (+212 % → ~+204 %) and −0.5 CPU-s. Not worth a
    root and a full gate now.
- **FFmpeg's +85 %** comes from the inline fast path at every check, and its speed needs them. Static placement s1 cuts the compile overhead from +85 % to +45 % at a 1.7 % speed loss; it, the other static rules and the profile hot list are closed (Table 6). Rows: `$EXTRA/wt-dev2-r/cct/btime.txt` (s9ff*), root `tsan-n1s-bd979a775aa7`.

## Notes

- Race preservation: 10 stock vs 10 optimized runs per app, no race-report site lost (memcached 4 kept, SQLite 3, MySQL 232-236, Redis all seeded; FFmpeg's stock reports none).
- memcached's workload V4 was chosen among five (2 Oct): default +2.8 %, pipelining +3.5 %, 32-key gets +6.3 %, long keys +16.8 %, the mix +30.4 %.
- Premises awaiting a ruling used by results of record: P-X86-FD (narrow), P-LIBEVENT, A5, A8, A9.
