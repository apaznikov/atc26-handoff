# TSan instrumentation optimizations: results

State: 5 Oct 2026. Every figure is the best sound result measured: a speedup over upstream TSan in the same leg,
4 code offsets, unless a cell says otherwise. `a` = AMD (2 × EPYC 9115), `f` = Intel (Xeon w9-3495X). Earlier, longer
versions are in the history (3abcd2c, b8d5c56).

Tables 1-6 hold every optimization with a result of any kind in the campaign's records (inventory of 5 Oct, about 180
items). Names in parentheses are the aliases the records use.

🟢 gain beyond the A/A control · 🟡 1-2 % or unresolved · ⚪ within ±1 % · 🔴 loss · — not measured

## Table 1. Summary (AMD)

| app | workload | submitted paper | sound, no annotations | sound + annotations backed by the program's own asserts | sound + our own annotations |
|---|---|---|---|---|---|
| SQLite | threadtest3: stress2, create_drop_index_1 | 1.71× | 1.02× ⚪ | **1.21×** (LO-OBJ-G) | — |
| FFmpeg | four transcodes of one film | 1.57× | **1.29×** (DynSTC-RT + N1 + N1-ST) | — | — |
| Redis | 7 data-heavy commands, 8 I/O threads | 1.45× | **1.45×** (FE-INL + N1 + quiet threads, derived automatically) | — | — |
| MySQL | Release, sysbench insert / update / delete, 24 connections | 1.16× | **1.13×** (FE-INL) | — | — |
| memcached | pipelined 32-key gets, 190-byte keys | 1.07× | 1.01× 🟡 over our stock TSan build (the analyses + EA-CONTENTS + SWMR-ROOTS) | — | **1.80×** (+ EVCONF line) |
| Chromium | — | 1.39× | not re-measured | — | — |

- **Annotations.** SQLite's spec is taken from SQLite's own `sqlite3_mutex_held` assertions (about 85 in btree.c).
  memcached's spec (EVCONF) states facts the program does not assert; it is checked by the compiler against the code
  and guarded at run time. Redis's quiet threads need no annotation: the compiler derives the 12 lines from the
  whole program (audits A60-A60c), with the same skips as the hand-written lines, which read 1.40× in the same leg.
- **FFmpeg** is carried by the single-threaded stream copy: copy 2.71×, mjpeg 1.04×, h264 1.00×, h265 0.99×.
- **Stock TSan over native:** SQLite 4.6× and ~19×, FFmpeg 2.8×, Redis 6.0×, MySQL 7.5×, memcached 6.7×.
- The submitted paper's figures came from other workloads, the Intel host and a compiler with two elisions later
  found unsound.

## Table 2. Optimizations that gain

The first eight rows are single levers over P1-v3 (the paper's analyses EA, LO, STC, SWMR, DE with every soundness
fix; a covered check is verified by an inline hit test). The rows from EA-CONTENTS down are measured on top of their
app's configuration.

| optimization | what it is | SQLite | memcached | Redis | MySQL | FFmpeg |
|---|---|---|---|---|---|---|
| **DynSTC-RT** | single-thread mode in the runtime: while one thread is alive nothing is recorded, range checks included | `a` ⚪ +0.4 | `a` ⚪ −0.3 | `f` 🔴 −2.2 | `a` ⚪ +0.1 | `f` 🟢 **+12.0** |
| **N1** | TSan's "already recorded?" test is inlined; the runtime is called only on a miss | `a` ⚪ −0.8 | `a` ⚪ +0.3 | `f` 🟡 +2.5 | `a` 🟡 −1.7 | `f` 🟢 **+5.0** |
| **N1-ST** | N1's inline test is skipped while the thread is in single-thread mode | `a` ⚪ +0.2 | `a` ⚪ −0.2 | `f` 🟡 −1.4 | `a` 🟡 −1.8 | `f` 🟢 **+16.9** |
| **FE-INL** | TSan's shadow call stack push and pop inlined | `a` ⚪ −0.7 | `a` ⚪ +0.8 | `f` 🟢 **+4.8** | `a` 🟢 **+3.6** | `f` ⚪ 0.0 |
| **FE-INL + VWIDE-loops** (DE-VERIFIED-WIDE) | plus run-time verified check removal inside loops | `a` ⚪ +0.1 | `a` ⚪ +0.6 | `f` 🟢 **+7.2** | `a` 🟢 **+4.6** | `f` 🟡 +1.8 |
| **FE-SINK** | the function-entry call moved to the first point that needs the frame | `a` 🟡 +1.1 | `a` 🟡 −1.7 | `f` 🟢 **+4.1** | `a` 🟢 **+2.8** | `f` 🟡 −1.2 |
| **N1-L** | N1 only in loops with at most 20 checks | `a` ⚪ +0.3 | `a` ⚪ −0.9 | `f` 🔴 −2.5 | `a` ⚪ −0.1 | `f` 🟢 **+3.0** |
| **MEMINTR** (MEMINTR-SRC, one-side) | a memcpy from a constant or private source has only its destination checked | `a` 🟡 +2.5 | `a` ⚪ +0.9 | `f` 🔴 −2.5 | `a` ⚪ +0.3 | `f` ⚪ 0.0 |
| **EA-CONTENTS** | a pointer read from a container no longer makes the container shared | — | `a` 🟢 **+1.4** | — | — | — |
| **SWMR-ROOTS** (SWMR-1) | no checks on reads of a global whose only write precedes every reader thread | — | `a` 🟢 **+0.9** | — | — | — |
| **EVCONF** (annotation) | objects owned by one event-loop thread are unchecked while a run-time guard holds | — | `a` 🟢 **+31.1** over upstream, with the two rows above | — | — | — |
| **EVCONF-RANGES** (annotation) | the guard applied to memset/memcpy on an owned object | — | `a` 🟢 **+18.5** | — | — | — |
| **EVCONF-ARGS** (annotation) | the confinement carried into the hash function's key argument (a clone of MurmurHash3) | — | `a` 🟢 **+7.3** | — | — | — |
| **EVCONF-INTERCEPT** (annotation) | the guard around memchr, strlen and bcmp on the confined read buffer | — | `a` 🟢 **+7.5** | — | — | — |
| **Quiet threads** (QUIET-THREADS, REDIS-MAIN, AUTO-BIO: silent-thread mode, the Redis phase guard) | Redis's main thread, which runs ≥ 99 % of the checks, skips its checks (range checks included) while every other thread is quiet since a release it acquired; the background threads' start is proven by whole-program rules | — | — | `a` 🟢 **+31.9** | — | — |
| **LO-OBJ-G** (annotation) | objects protected by their owner's lock are unchecked while the thread holds that lock | `a` 🟢 **+20.9** over upstream (spec v7) | — | — | — | — |

- **Quiet threads:** the rule that admits only park objects and start routines it can check alone (AUTO) reads
  0.946-0.962 of the best on Redis; the whole-program derivation of bio's lines (AUTO-BIO) gives +31.9 %.
- N1-ST is measured on top of N1 + DynSTC-RT; FE-SINK adds nothing on top of FE-INL. On FFmpeg the paper's
  compile-time DynSTC gives +10.1 %, DynSTC-RT +12.2 %, both together +31.9 % (DYNSTC-DIRECT); the compile-time one
  loses elsewhere (Redis 0.945, memcached 0.995, MySQL 0.993).

## Table 3. No gain

| idea | what it is | result |
|---|---|---|
| Removal-mode DE (DE-REMOVAL), DE-3R, DE-2R | covered checks deleted outright; one range check per loop; adjacent fields merged | ⚪ MySQL +0.3 %, SQLite 0, memcached 0, FFmpeg −0.3 % (full leg; a screening had read +10.2 %); 🔴 Redis −3.8 % |
| **All-paths DE + cycle cut** (DE-ALLPATHS, DE-5) | a check is covered if every path to it has a cover, even when none dominates; covers kept around loops | ⚪ Redis +1.0 %, SQLite −0.6 %, MySQL +0.1 %, memcached −1.2 %; ≤ 1.5 % of checks |
| Loop guard (T8, DE-LC) and the T10 package | a loop-invariant access checked once per synchronisation-free stretch; the older full lever package | ⚪ Redis +2.2 / +3.1 %, memcached 0 / +1.6 %, MySQL −0.1 / −4.2 %, SQLite unresolvable; FFmpeg loop guard +1.4 % |
| Levers on top of SQLite's LO-OBJ-G | N1, N1-L, MEMINTR, FE-INL | ⚪ FE-INL +4.5 %, MEMINTR +3.5 % (inside a wide A/A), N1 🔴 −18.5 % |
| The old "same location" rule (SAME-LOCATION, DSL) | struct fields cover each other (S), array elements cover each other (A), a narrower check covers a wider one (Z) | 🔴 unsound: S loses a real memcached race. Upper bound only: Redis +11 %, SQLite +31 %, memcached +7 %, MySQL +8 % (table 5) |
| LO-OBJ-ARGS | LO-OBJ-G's ownership carried into SQLite's record comparison | ⚪ unsound ceiling 1.023, inside the A/A |
| LO-OBJ-RANGES | LO-OBJ-G's guard on SQLite's memcmp and VDBE memcpy | ⚪ reaches neither site |
| EA interceptor toggle in per-unit builds | EA's libc-call toggle without the whole-program definitions list | ⚪ SQLite 0.977, MySQL 1.003 |
| memcached's flag globals | unlocked reads of `settings`, `expanding`, `hashpower` | unsound ceiling +4.5 %; no sound route (admin-written; a genuine benign race) |
| Redis I/O threads' client buffers | the I/O threads' checks on the buffers they drain | ⚪ ≈ 0: their cycles are a spin, not checks |
| Quiet mode beyond Redis | the phase guard on the other four apps | ⚪ ≤ 0.4 % of checks fall in quiet intervals |
| MySQL THD owner reads (THD-READS, X3) | a connection's own THD fields read by its thread | ⚪ ≤ 2.35 % of checks |
| Run-time owner tag for one-thread objects | skip the owner's accesses to memory only one thread touches | unsound: a skipped access leaves no record for a later foreign access |
| N1-ATOMIC on SQLite and MySQL | inline test for relaxed atomics | ⚪ ≤ 0.65 % of cycles |
| DE across synchronisation-free calls (nosync census) | DE's "call between" class on whole-program IR | ⚪ Redis 0.51 %, SQLite 0, MySQL ≤ 2.08 % of checks |
| DE-5, DE-6, DE-7, DE-8 | finer rules for when a call or a cycle breaks a cover | ⚪ −0.8…+1.4 % |
| IPA-DE | a check in a callee covers one in its caller | ⚪ ≤ 1 % of checks |
| VWIDE, VWIDE-loops alone | run-time verified removal where no check dominates | ⚪ −0.9…+1.3 % |
| FE-hot, hot-list N1 (PGO-PLACE), N1-S, N1-LOOPS-∞ | FE-INL or N1 only at hot sites | 🔴 none beats the full version |
| FE-PM, N1-PM, N1b | out-of-line entries that save fewer registers | 🔴 MySQL −2 %, SQLite −2…−4 % |
| FE-LAZY, FE-INL-CSE | entry recorded only when needed; one thread-state load per function | ⚪ ±1 % |
| N1-CSE, N1-ATOMIC, LIBCALL-INLINE, N2 | compact, atomic, libc-call and batched variants of the check | ⚪ ±1 %; N2 as built loses races; N1-ATOMIC on Redis +1.6 %, unresolved |
| SUBS, SUBS-SEL | a covering record of the same thread counts as a hit | 🔴 −2…−18 %; the selective form is unsound |
| Whole-program summaries (STC-SUM), STC-TS, allowlists (STC-WL, libfacts) | closed-world facts for STC and SWMR | ⚪ < 1 % of checks |
| LTO, Attributor, extra LLVM passes (XPASS, XP2) | more optimization before instrumentation | ⚪ nothing; the Attributor miscompiles |
| CLONE-ESC, EA-SLOT (EA-SLOT-CONTENTS), ICALL-A2, returns-fresh (named allocators), field chase | finer escape analysis | ⚪ each < 2 % of checks |
| MySQL whole-program mode (MYSQL-WP); top-down parameter facts (TOPDOWN-FACTS) | summaries for MySQL; "argument local in every caller" across units | ⚪ ≤ 0.14 % of checks |
| Per-field and heap SWMR (SWMR-G, PUBLISH-ONCE), thread roles, thread ids by creation history (THREAD-IDS) | finer may-happen-in-parallel facts | ⚪ each < 3 % of checks |
| NOALIAS, custom lock wrappers (CUSTOM-SYNC), MySQL sysvars, InnoDB latches (INNODB-LATCH), ODR trust (ODR-TRUST) | language and library facts | ⚪ each < 3.2 % of checks |
| RT-SYNC, SLOT-CHURN (SID-ALIAS), RANGE-OVERWRITE | runtime-only changes | ⚪ no effect |
| LO-F, LO-W, LO-B1/B2, sound loop ranges, DE-1, DE-2, DE-4, DE-9 | further lock-ownership and DE rules | ⚪ ≤ 1.4 % of checks, or unsound |

## Table 4. Open

| item | app | status |
|---|---|---|
| **memcached EVCONF without the spec**: the spec derived from libevent's event-base affinity, the owner store and the foreign-reader closure | memcached | in work: a report-only derivation first |
| **EVCONF-FIELDS**: the owner's reads of its connection's own fields | memcached | `a` +3.6 % on top of table 1; audited; needs the wider P-X86-FD |
| QUIET-FE: function entry and exit not recorded while Redis's main thread skips | Redis | FE is 10.3 % of the main thread's cycles; not built |
| DE-AV (SAME-PTR, DE-10): DE's covers checked at run time where no dominance holds | SQLite | unsound ceiling +8-11 %; a sound run-time flag reaches 3-17 % of executions, about +2-4 % expected; parked |
| LO-OBJ-G on the connection's objects under `db->mutex` | SQLite | unsound ceiling +5.8 %; needs a second lock type; parked |
| Camera-ready legs | all | trees built on the one camera-ready compiler and IR-identical to the measured ones; runtime: Redis on the quiet-mode runtime, the others on the runtime without the guard (its hit-path test costs MySQL 2 % on AMD; with the TLS fix alone MySQL reads 1.134 over upstream); held for the go |

## Table 5. Where the analyses are conservative (shares of executed checks)

| analysis | what it cannot prove | SQLite | memcached | Redis | MySQL | FFmpeg |
|---|---|---|---|---|---|---|
| DE, dominance | the same address checked twice with no acquire between, the earlier dominating, both kept | 3.35 % | 0.04 % | 3.44 % | 5.45 % | 5.92 % |
| of which: a call between | the callee is not proven synchronisation-free (external or indirect / local) | 1.67 / 0.69 % | 0.04 / 0 % | 3.38 / 0.00 % | 2.78 / 1.06 % | 3.53 / 0.57 % |
| of which: the address question | only the equality of the two addresses is unproven | 1.0-1.7 % | 0 | 0.06 % | 1.6-2.7 % | 1.8-2.4 % |
| DE, post-dominance | the later check post-dominates; a call breaks it / ceiling with calls allowed | 0.28 / 0.37 % | 0 / 0 % | 0.30 / 0.92 % | 0.02 / 0.53 % | 0.21 / 0.53 % |
| DE, availability | neither check dominates the other | 4.36 % | 0.08 % | 5.79 % | 1.00 % | 8.56 % |
| DE, all paths / cycle cut | a cover on every path; a cover lost around a loop | 0.22 / 0.92 % | 0 / 0.71 % | 0.82 / 0.14 % | 0.07 / 0.08 % | 0.5 / 0.89 % |
| DE, stronger alias analysis (DE-AA) | must-alias from SCEV or points-to | 0 | 0 | 0 | — | 0 |
| EA, all (EA-TL) | memory one thread touches or consistently ordered, kept by EA | 72 % | 55 % | 63 % | 49 % | 70 % |
| EA, pointer parameter | reached through a pointer argument (share of the row above) | 53-75 % | 53-75 % | 53-75 % | 76 % | 53-75 % |
| EA, cross-unit parameter facts | "argument local in every caller", upper bound | 0.04 % | 0.05 % | 0.00 % | 0.14 % | 0.01 % |
| EA, own stack | the thread's own stack, no other thread touches it | 4.9 % | 5.9 % | 12.5 % | 14.8-24 % | 5.8 % |

- Most of the one-thread mass is real at run time but not provable statically: it is heap memory reachable from shared
  structures (EA-HEAP: 78.7 / 7.5 / 34.0 / 41.9 % of the checks of SQLite / FFmpeg / memcached / Redis; the part freed
  in the allocating call ≤ 0.6 %).
- **The old "same location" rule** (unsound reference, removal mode, over exact DE in the same leg; S + A + Z over
  upstream TSan in the last row):

  | variant | Redis (`f`) | SQLite (`a`) | memcached (`f`) | MySQL (`a`) | FFmpeg, checks removed |
  |---|---|---|---|---|---|
  | S: fields of one struct | +8.4 % | +11.9 % | +1.7 % | +4.1 % | 13.7 % |
  | A: elements of one array | +5.9 % | +12.8 % | +3.9 % | +1.1 % | 22.1 % |
  | S + A | +11.5 % | +27.3 % | +6.6 % | +6.0 % | 34.4 % |
  | S + A + Z (narrow covers wide) | +10.6 % | +31.2 % | +6.8 % | +7.9 % | 35.4 % |
  | S + A + Z over upstream | +11.3 % | +35.4 % | +7.3 % | +10.3 % | — |

  Z alone removes nothing. S loses a race stock reports in every run: memcached's `do_item_link` writes `curr_bytes`
  then `curr_items` under the stats lock, `clock_handler` reads `curr_items` without it, and under S the first write
  covers the second. A and exact DE lose none. The shadow-proxy rule (RedCard), which reports a different race than
  stock, would remove only 0.1-1.1 % of checks.

## Table 6. Every other optimization tried

| optimization (aliases) | what it is | result | standing |
|---|---|---|---|
| **DE** | | | |
| DE package (T4, T4+E) with DE-B selective peeling | merge + loop ranges + selective peeling | with exact DE: memcached +3.6 % (`f`), MySQL +3…+6 %, Redis −2.6 %, SQLite and FFmpeg within ±1 %; DE-B reaches ≤ 0.22 % | superseded |
| N3, N4, N8 | DE path flags; covers across iterations; trimmed entry/exit | ≤ 1 % each | closed |
| DE-8b | C allocation functions treated as synchronisation-free | 0 %, and unsound | dropped |
| relaxed-atomic rule (`-tsan-de-relaxed-atomic-nosync`) | relaxed atomics do not break a cover (A8) | SQLite's loop-guard skips 0.74 → ≈ 9.2 % of checks | on in every build |
| pure-asm rule | inline asm treated as synchronisation-free | ≈ 0 | closed |
| anticipation, acquire tolerance, available-checks dataflow | further cover placements | ≤ 0.56 % | closed |
| POSTDOM-TERM | post-dominance with proven termination | ≤ 0.27 % | in the tree |
| **EA** | | | |
| U2 | unsound bound: named arguments never escape | 0.963-0.996 over T1 on four apps | closed |
| integer-copy closure off (A7) | EA without following pointers copied as integers | memcached +0.9, Redis +1.0, MySQL −2.7 % | closure kept on |
| EA-7; the record-compare chain | arguments of external or address-taken functions | +5.5 k static sites; unsound bound ≈ 5.6 % of SQLite's checks | shelved |
| EA-WP | EA with whole-program summaries | 0 % of the one-thread heap mass | closed |
| EA call-site flags (MAAP-style), OWN-STACK-ENTRY (X2) | per call site, pointer-parameter checks removable | MySQL: 0.01 % with nocapture, 3.01 % upper bound; own-stack entry test ceiling 24.4 %, unsound without an escape argument | parked 5 Oct |
| **STC, SWMR, LO** | | | |
| STC-1, STC-2, STC-3, STC-4, join-aware STC, STC-CALLEE, STC-STRONG (i) (O-STC), DYN-1 | finer single-threaded-context rules | ≤ 2.44 %, mostly ≈ 0 | closed |
| MAIN-ONLY (STC-STRONG (ii)) | prove statically that Redis's reads happen only on the main thread | statically impossible; 22 % dynamic ceiling, taken at run time by the quiet threads | closed |
| DYN-2 | where DynSTC's guard skips | only FFmpeg's stream copy | closed |
| LO-unwind | an invoke's unwind edge inherits the callee's acquisitions | refuted | closed |
| SQLITE-CW | closed world for SQLite | 0.02 % | closed |
| LO-C | POSIX calls transparent to LO | no gain | closed |
| SWMR-H | a location written only before its publication | ceiling ≤ +0.2 %; 14 lost-race paths in audit | parked |
| SWMR-ROOTS on SQLite, MySQL, Redis | memcached's rule on the other apps | ≤ 2.9 % of checks, ≈ 0-1.3 % reachable | closed 5 Oct |
| G-EA | escape analysis under LO-OBJ-G's guard | loses 3.7 % | closed |
| LO-OBJ-G spec completion | BtShared fields spec v7 does not name | ≈ 1.2 % of one subtest's checks | not built |
| LO-OBJ-G Pager/Wal roots | Pager and Wal under their b-tree's mutex | 12.7 % of the locked mass | declined (P-PAGER) |
| SQLITE-KEY | comparator key and payload in the spec | ≈ 3 % of checks | parked |
| SQLite sharable guard | a per-object guard for the shared cache | cannot reach the mass | closed |
| PERELEM | per-element locksets | memcached ≈ 2.3 %, provable ≈ 0 | parked |
| TLS-rooted escape analysis (LO-TID) | objects reachable only from a `thread_local` root | MySQL ≈ 0.8 % on Release | parked |
| **N1, FE and other inline paths** | | | |
| N1-ST =miss | the flag read only on a miss | FFmpeg −6.6 % against the front form | front kept |
| N1-ST (b), (c) =fs | the flag hoisted; the flag in the fast state | equals the front form | front kept |
| N1-ST-WORKER | the front test dropped in worker-only code | 0.00 % | closed |
| N1-SPLIT, N1-PAIR | cold miss blocks; one load tests two cells | 0; subsumed by DE-2R | closed |
| N1-CSE across calls | sibling shadow tests shared across calls | ≤ 1 % expected | not built |
| FE-INL exit-max=1; unified exit | FE-INL variants | −5.2 % on MySQL; identical | dropped |
| FE oracle; TSan-aware inlining | entry/exit removed (unsound ceiling) | Redis +23.5 pp, MySQL +13.6, SQLite +4.4 | ceiling |
| call-cost ceiling (N1/N2-real) | run time in the runtime calls of hitting checks | SQLite 21 %, FFmpeg 17.5 %, Redis 25 %, memcached ≈ 0 | bounded N1, N2 |
| FE-NOFRAME | no entry/exit in functions without accesses | — | not taken: changes report stacks |
| MEMINTR-INLINE | small memcpy/memset inlined | ≤ 1 % | not recommended |
| **Annotation-based and ownership variants** | | | |
| EVCONF-INTERCEPT write side (X5) | the guard on sendmsg/writev iovecs | unsound ceiling +0.7 % | closed |
| fresh item until linked; one-sided `_nosrc` copies | memcached's new item; covered-source copies | ≤ 1.1 %; ≤ 0.8 % | not built |
| quiet-thread refinements | the skip bit folded into the fast-state load; re-read only after synchronising calls | ≤ 0.3 % beyond the fold | parked |
| R1-R7 | the EVCONF full-connection claim, quiet-mode allocation stacks, skipping the stats mutex, a SQLite connection latch, refcount readers, DOM-FS, guard hoist | unsound, report-changing, or ≤ 0.3 % | rejected 4 Oct |
| OWN-HANDOFF | checks skipped while the program's hand-off protocol says one thread holds a buffer | Redis +13.9 %; trusts the program's protocol | not adopted; the quiet threads replace it |
| OWN-CONN | an owner guard on SQLite's private connections | ≈ +21 % estimated | withdrawn (P-CONN not asserted) |
| OWN-STACK | checks skipped on the thread's own stack | sound form ≈ −1 % | parked |
| SPIN-ACQ | a spin of atomic loads acquires only on a change | Redis ≤ +14 % (ceiling) | parked |
| FFmpeg heap hand-off | buffers handed between FFmpeg's threads | rests on a program-protocol premise | closed |
| **Runtime-only track** (parked from the paper) | | | |
| DD-EXACT (DD-COST; A = DD-CHEAP, B1 = DD-BIG) | TSan's deadlock detector | A sound, null; B1 memcached +21.7 % but drops lock-order reports | parked |
| RT-SYNC-LF | lock-free acquire loads | unresolved | parked |
| RT-RANGE, RANGE-HIT-SKIP, RANGE-VEC, RANGE-UNIFORM | faster range checks | ≤ 0.5-2 % | closed or parked |
| RT-ALLOC | faster allocation bookkeeping | Redis ≈ 11 % of cycles (mass) | parked |
| RT-SLOT, SLOT-PREF, empty-epoch elision, early reset | slot preemption and global resets | ≤ 1.7 % | closed |
| ALLOC-COVER, RANGE-COVER | allocation and range records covering later checks | 5.6-7.0 % (proxy) | out of scope |

No result yet: sendmsg batching, the interceptor bypass, EA-P1's out-parameter cost, small EA precision counts.

## Notes

- **Method.** Each arm is built at four code offsets (0/16/32/48 bytes mod 64); each leg has an A/A arm (the base
  run twice); a gain counts when it lies beyond the A/A range at every offset. Servers and clients run on disjoint CPUs.
- **Baseline.** Upstream TSan = the LLVM fork point plus upstream's fix for unchecked fields of escaped locals
  (`tsan-pristine-up59b2-0f1aed148576`), measured in the same leg.
- **memcached's workload** was chosen among five measured over upstream (2 Oct configuration): default input
  +2.8 %, pipelining +3.5 %, 32-key gets +6.3 %, long keys +16.8 %, the chosen mix +30.4 %.
- **Camera-ready compiler:** `tsan-cr-8345a0396fa5`, one compiler for all apps (audits A42-A58b; IR suites, quiet-mode
  tests, check-tsan in 12 configurations, go-check).
- **Race preservation.** Every row of tables 1-2 loses no race stock TSan reports under the premises in
  `soundness-fixes.md` §6: IR tests, check-tsan, reproducers with controls, an independent audit, and application runs
  against stock (10 runs per arm, races matched by location pair: memcached 4 kept, SQLite 3, MySQL 232-236, Redis all
  seeded races; FFmpeg's stock reports none).
- **Premises awaiting a ruling:** P-X86-FD (narrow: EVCONF; wider: EVCONF-FIELDS), the libevent callback contract,
  MALLOC-ATTR, A5, A8, A9.
- **Compile time** of the paper's analyses: SQLite +20 %, memcached +8 %, Redis +9 %, FFmpeg +16 %.
