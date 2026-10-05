# TSan instrumentation optimizations: results

State: 5 Oct 2026, 12:10. Speedups are over upstream TSan measured in the same leg ("direct") unless a cell says
otherwise. `a` = AMD (2 × EPYC 9115), `f` = Intel (Xeon w9-3495X). Method, baseline and race preservation: Notes.
Longer earlier versions are in the history (commits 3abcd2c, ec0d58e).

**Completeness.** Tables 1-6 hold every optimization that has a result of any kind (a timing, an unsound ceiling, a
count of its reach, or a closure with its reason) in the campaign's records: `ideas.md`, `results-matrix.md`, the
tracker `experiments.md`, the runtime track and every earlier version of this file. The inventory of 5 Oct counted
about 180 items in two independent passes. A name in parentheses is the alias the records use.

🟢 gain resolved above the A/A control · 🟡 unresolved, or 1-2 % · ⚪ within ±1 % · 🔴 loss of 2 % or more · — not measured

## Table 1a. All applications, without annotation-based optimizations

| app | workload | stock TSan over native | configuration | AMD | Intel | submitted paper |
|---|---|---|---|---|---|---|
| FFmpeg | four transcodes of one film | 2.8× | DynSTC-RT + N1 + N1-ST | 🟢 **+29.2 %** (1.285-1.305, 4 offsets) | 🟢 **+30.5 %** (1.290-1.314, 4 offsets); +34.0 % over our stock arm | +57 % |
| Redis | seven data-heavy commands (LRANGE_100/300/500/600, MSET, ZADD, ZPOPMIN) | 6.0× | FE-INL + N1 | 🟢 **+7.4 %** (1.055-1.096, 4 offsets; 8 I/O threads, server and client on disjoint CPUs); 1.035-1.099 in four later legs | 🟢 **+5.7 %** (1.043-1.073, 4 offsets; 12 I/O threads, disjoint CPUs); 1.043-1.056 in three later legs | +45 % |
| MySQL | Release build, sysbench insert, update_non_index, delete; 24 connections, server and client on disjoint CPUs | 7.5× | FE-INL on AMD; the paper's analyses alone on Intel | 🟢 **+9.4 %** (4 offsets); +12.4 % and +12.6 % in two later legs on root 1738 (3 and 5 Oct); +5.4 % on shared CPUs | ⚪ −0.2 %; with FE-INL 🔴 −3.0 % (0.968-0.971), −2.7 % in a later leg | +16 % (select), +11 % (write-only) |
| memcached | pipelined 32-key gets with 190-byte keys (V4: pipeline 32, multi-key get 32); server and client on disjoint CPUs | 6.7× | the paper's analyses + EA-CONTENTS + SWMR-ROOTS | 🟡 +1.0 % over our stock arm (1.004-1.015, A/A 0.995-1.002); −1.7 % over upstream (derived) | 🟡 +3.2 % over our stock arm (4 offsets); about +1.2 % over upstream (derived) | +7 % |
| SQLite | threadtest3, shared-cache subtests stress2 and create_drop_index_1 | 4.6× and ~19× | the paper's analyses | ⚪ +2.3 % (0.976-1.073, inside the A/A; 4 offsets); −1.8 % over our stock arm | — | +71 % |

- **FFmpeg** is the geomean of four transcodes and is carried by the single-threaded stream copy: over upstream, copy
  2.71× (AMD) / 2.88× (Intel), mjpeg +4.2 % / +3.7 %, h264 −0.2 % / −1.7 %, h265 −1.1 % / −1.1 %. The two encoders
  run mostly in uninstrumented x264/x265 code (stock TSan costs them 1.3-1.4×), and the inline single-thread test
  costs them slightly.
- **Between legs** the Redis and MySQL ratios move by about ±3 %, more than the within-leg A/A. The camera-ready legs
  run headline rows at N=2 per offset and pool them.
- **The last column** is what the submitted paper printed for all its analyses together: other workloads, the Intel
  host, and earlier compilers with two elisions later found unsound.

## Table 1b. Applications where annotation-based optimizations gain

A short per-application description names objects and their owner (memcached, SQLite) or the other threads' park
objects (Redis). The compiler checks it against the code, and a run-time guard tests the condition. Workloads as in
table 1a; premises in `soundness-fixes.md` §6.

| app | configuration | AMD | Intel | standing |
|---|---|---|---|---|
| memcached | EVCONF + SWMR-ROOTS + EA-CONTENTS + EVCONF-RANGES + EVCONF-ARGS + EVCONF-INTERCEPT, with IN-BOUNDS (tree cwn) | 🟢 **+80.3 %** (1.792-1.812, 4 offsets, A/A 0.996-1.002) | 🟢 **+58.5 %** (1.571-1.613, 4 offsets, A/A 0.995-1.013) | audits A46-A58b; preservation: 0 races lost; two of EVCONF's premises await a ruling (Notes) |
| Redis | FE-INL + N1 + **quiet threads** (QUIET-THREADS, REDIS-MAIN: silent-thread mode, the Redis phase guard), annotated, range skip on (tree qpr; camera-ready qxr is IR-identical) | 🟢 **+38.7 %** (1.349-1.452, 4 offsets, A/A 0.980-1.003); +34.1 % in an earlier leg on the final root | 🟢 **+31.4 %** (4 offsets, A/A 0.987-1.033) | annotations: 12 compiler-checked lines on Redis's background thread (its start routine and helpers, two lists under its mutex, two allocators, the configuration table); without them the same guard reads 0.946-0.962 of the best. Audits A43, A45-A57c; preservation: no seeded race lost; premises FIELD-ADDR, A12-DATA, P-DICT adopted |
| SQLite | LO-OBJ-G | 🟢 **+23.5 %** with spec v6 (1.180-1.302, 4 offsets × N=4; stress2 +23.9 %, create_drop_index_1 +23.1 %; +18.5 % over our stock arm, the faster base here); spec v7 (with the latch) +20.0 % and +20.9 % in two legs | — | v7 audited; the camera-ready leg carries v6 beside v7 |

- **memcached's levers, V4, AMD / Intel:** EVCONF-RANGES +18.5 / +13.9 %, EVCONF-ARGS +7.3 / +7.2 %, EVCONF-INTERCEPT
  +7.5 / +5.7 %. The configuration before them reads +31.1 / +23.3 % over upstream in the same direct legs, and their
  product matches the direct ratio (AMD 1.367 against 1.375, Intel 1.290 against 1.286). EVCONF-FIELDS would add
  +3.6 / +2.9 % (table 4).
- **Redis's quiet threads (silent-thread mode) over FE-INL + N1 in the same leg:** AMD +26.3 % (earlier leg +29.6 %), Intel +25.6 % (other
  legs +27.4 %, +31.4 %); the range skip's own share is +3.6…+9.9 %. The unsound ceiling of skipping all of the main
  thread's plain checks is +36.5 % (Intel). The gain does not depend on the I/O thread count: with 4 and 6 I/O threads
  instead of 12 (range skip off) the annotated arm reads 1.277 and 1.265 over the best.
- **One configuration for every app** (derived): N1 + N1-ST + DynSTC-RT gives FFmpeg +24 %, Redis +6.5 %, and loses on
  SQLite (−0.5…−3.6 %), memcached (−6.7 %) and MySQL (−2.6 %).
- **The paper's analyses alone** (P1-v3, below) are not separable from stock on any app (0.98-1.02). Each analysis
  alone (tier B, N=3, 25 Sep: EA, LO, STC, SWMR, DE, DE with loop peeling, the compile-time DynSTC) reads −5.5…+2.1 %
  over stock on SQLite, Redis, MySQL and memcached; loop peeling adds nothing.
- **Ceilings for removing checks** (profile oracles: memory touched by one thread, O1-all / also Eraser-consistent, O2-eraser): SQLite
  +16 / +53 %, memcached +6 / +11 %, Redis +17 / +27 %, MySQL −1 / +3 %, FFmpeg +31 / +41 %. They come from 26 Sep
  profiles of older workloads and count memcached's request keys and connection fields as shared, which is why the
  annotations exceed them. Partial oracles: one-thread heap only (O1-heap) SQLite +8.8 %, Redis +7.1 %; lock-consistent
  only (O2-strict) 0.986-1.004 on SQLite, memcached, MySQL and Redis.

## Table 2. Optimizations that gain

Single-lever columns are over the base P1-v3 (the paper's analyses EA, LO, STC, SWMR and DE with every soundness fix;
a covered check is verified by an inline hit test instead of being removed). The rows from EA-CONTENTS down are
measured on top of their app's configuration of record (table 1).

| optimization | what it is | SQLite | memcached | Redis | MySQL | FFmpeg |
|---|---|---|---|---|---|---|
| **DynSTC-RT** | single-thread mode in the runtime: while one thread is alive nothing is recorded, range checks included | `a` ⚪ +0.4 | `a` ⚪ −0.3 | `f` 🔴 −2.2 | `a` ⚪ +0.1 | `f` 🟢 **+12.0** |
| **N1** | TSan's "already recorded?" test is inlined; the runtime is called only on a miss | `a` ⚪ −0.8 | `a` ⚪ +0.3 | `f` 🟡 +2.5 | `a` 🟡 −1.7 | `f` 🟢 **+5.0** |
| **N1-ST** | N1's inline test is skipped while the thread is in single-thread mode (flag read before the test) | `a` ⚪ +0.2 | `a` ⚪ −0.2 | `f` 🟡 −1.4 | `a` 🟡 −1.8 | `f` 🟢 **+16.9** |
| **FE-INL** | the push and pop of TSan's shadow call stack are inlined instead of calling the runtime | `a` ⚪ −0.7 | `a` ⚪ +0.8 | `f` 🟢 **+4.8** | `a` 🟢 **+3.6** | `f` ⚪ 0.0 |
| **FE-INL + VWIDE-loops** (DE-VERIFIED-WIDE) | plus run-time verified check removal inside loops | `a` ⚪ +0.1 | `a` ⚪ +0.6 | `f` 🟢 **+7.2** | `a` 🟢 **+4.6** | `f` 🟡 +1.8 |
| **FE-SINK** | the function-entry call is moved to the first point that needs the frame | `a` 🟡 +1.1 | `a` 🟡 −1.7 | `f` 🟢 **+4.1** | `a` 🟢 **+2.8** | `f` 🟡 −1.2 |
| **N1-L** | N1 only in loops with at most 20 checks | `a` ⚪ +0.3 | `a` ⚪ −0.9 | `f` 🔴 −2.5 | `a` ⚪ −0.1 | `f` 🟢 **+3.0** |
| **MEMINTR** (MEMINTR-SRC, one-side) | a memcpy from a constant or private source has only its destination checked | `a` 🟡 +0.7…+2.5 | `a` ⚪ +0.9 | `f` 🔴 −2.5 | `a` ⚪ +0.3 | `f` ⚪ 0.0 |
| **EA-CONTENTS** | a pointer read from a container no longer makes the container shared | — | `a` 🟢 **+1.4** (on against off) | — | — | — |
| **SWMR-ROOTS** (SWMR-1) | no checks on reads of a global whose only write precedes every reader thread | — | `a` 🟢 **+0.9** on top of EVCONF | — | — | — |
| **EVCONF** (annotation) | objects annotated as owned by one thread are unchecked while a run-time guard holds (no idle-timeout thread, no external storage, connection never lent) | — | `a` 🟢 **+22.7** over stock on shared CPUs; +32.2 with the two rows above on disjoint CPUs | — | — | — |
| **EVCONF-RANGES** (annotation) | EVCONF's guard applied to memset/memcpy on an owned object (constant length in its type, or any length under IN-BOUNDS); one memset zeroing each response object was 16.6 % of the cycles | — | `a` 🟢 **+18.5**, `f` 🟢 **+13.9** (V4; AMD V3 +10.3); Intel equals the unsound ceiling (+14.1) | — | — | — |
| **EVCONF-ARGS** (annotation) | EVCONF's confinement carried into the hash function's key argument: a clone of MurmurHash3 with the key reads unchecked while the guard holds, called only where the key is a covered request key | — | `a` 🟢 **+7.3**, `f` 🟢 **+7.2** (V4; AMD V3 +5.1); unsound ceiling of all key reads +8.2 (Intel) | — | — | — |
| **EVCONF-INTERCEPT** (annotation) | EVCONF's guard around the libc calls that scan the confined read buffer: memchr, strlen and the request-key side of bcmp | — | `a` 🟢 **+7.5**, `f` 🟢 **+5.7** (V4, 4 offsets); unsound ceiling +6.5 (Intel) | — | — | — |
| **Quiet threads** (QUIET-THREADS, REDIS-MAIN: silent-thread mode, the Redis phase guard; annotation) | Redis's main thread, which runs ≥ 99 % of the checks, skips its checks (range checks included) while every other thread is quiet since a release it acquired; annotations register the threads' park objects and attest their start | — | — | `a` 🟢 **+26.3**, `f` 🟢 **+25.6** | — | — |
| **LO-OBJ-G** (annotation) | objects annotated as protected by their owner's lock are unchecked while the thread holds that lock | `a` 🟢 **+18.5** over stock (both subtests, 4 offsets; the first +18.8 read stress2 only) | — | — | — | — |

- N1-ST is measured on top of N1 + DynSTC-RT and costs 1-3 % on multi-threaded code. FE-SINK is measured over
  P1-v3 + N1 and adds nothing on top of FE-INL.
- FFmpeg, one leg over stock: the paper's compile-time DynSTC +10.1 %, DynSTC-RT +12.2 %, both together +31.9 %,
  the configuration of record +33.2 % (DYNSTC-DIRECT). The compile-time DynSTC loses elsewhere: Redis 0.945, memcached 0.995, MySQL
  −0.7 %. The inline guard skips plain accesses, the runtime mode skips range checks;
  each is about half of the single-threaded work.
- Re-timed on the workloads of record (2 Oct): no other lever adds to memcached's or Redis's configuration.

## Table 3. No gain

| idea | what it is | result |
|---|---|---|
| Removal-mode DE (DE-REMOVAL), DE-3R, DE-2R | covered checks deleted outright; one range check per loop; adjacent fields merged | ⚪ MySQL +0.3 %, SQLite 0, memcached 0; 🔴 Redis −3.8 %; FFmpeg open (table 4). Relaxed screenings: FFmpeg DE-2R +3.7 %, both +6.9 % (29 Sep); MySQL Debug +3.2 / +6.8 % |
| **All-paths DE + cycle cut** (DE-ALLPATHS, DE-5) | a check is covered if every path to it has a cover, even when none dominates; covers kept around loops | ⚪ over the best, AMD, 4 offsets: Redis +1.0 %, memcached −1.2 % (Intel), SQLite −0.6 %, MySQL +0.1 %, all inside their A/A (3 Oct); ≤ 1.5 % of checks |
| Loop guard (T8, DE-LC) and the T10 package | loop guard: a loop-invariant access is checked once per synchronisation-free stretch and re-checked when the shadow generation moves; T10: the older compiler's full lever package (ALL, with N1-L) including it | ⚪ over the paper's analyses, 4 offsets (4 Oct): Redis +2.2 / +3.1 %, memcached 0 / +1.6 %, SQLite not resolvable (A/A ±7 %), MySQL −0.1 / 🔴 −4.2 %. FFmpeg (25-26 Sep): loop guard v2 +1.4 % (mjpeg +5.3 %); T10 +20.5 % with the inexact DE of the time (mjpeg and copy 1.44), +9.9 % with exact DE |
| Earlier levers on SQLite v7 | N1, N1-L, MEMINTR, FE-INL on top of the configuration of record | ⚪ FE-INL +4.5 %, MEMINTR +3.5 % at the edge of a wide A/A (0.974-1.053), N1 🔴 −18.5 %; FE-INL and FE-INL + MEMINTR are arms of SQLite's camera-ready leg |
| The old "same location" rule (SAME-LOCATION, DSL; unsound reference) | struct fields cover each other (S), array elements cover each other (A), a narrower check covers a wider one (Z) | 🔴 unsound: S loses a real memcached race. Measured as the related-work bound: Redis +6-12 %, SQLite +12-31 %, memcached +2-7 %, MySQL +1-8 % (table 5 notes) |
| LO-OBJ-ARGS | LO-OBJ-G's lock ownership carried into SQLite's record comparison and the schema-name compares | ⚪ unsound ceiling 1.023 (AMD, 4 offsets × N=4), every offset inside the A/A (0.972-1.063); closed 5 Oct |
| LO-OBJ-RANGES | LO-OBJ-G's guard applied to SQLite's memcmp and VDBE memcpy | ⚪ reaches neither site: the copied registers are read without `db->mutex`, and the page side arrives through a function pointer (4 Oct) |
| EA interceptor toggle in per-unit builds | EA's libc-call toggle without the whole-program definitions list (A4 adopted) | ⚪ SQLite 0.977 (A/A 0.907-0.983), MySQL 1.003 (A/A 0.992-1.001); closed 5 Oct |
| memcached's flag globals | the unlocked reads of `settings`, `expanding` and `hashpower` | unsound ceiling +4.5 % (Intel); no sound route: admin commands write `settings`, and `expanding` is a genuine benign race; closed 4 Oct |
| Redis I/O threads' client buffers | skipping the I/O threads' checks on the buffers they drain | ⚪ ≈ 0: 98.5 % of the I/O threads' cycles are a spin on `io_threads_pending`, not checks; closed 4 Oct |
| Quiet mode beyond Redis | the phase guard on the other four apps | ⚪ ≤ 0.4 % of checks fall in quiet intervals (census, 4 Oct) |
| MySQL THD owner reads (THD-READS, X3) | a connection's own THD fields read by its thread, under an owner guard | ⚪ ≤ 2.35 % of plain checks, below the 3 % bar (4 Oct) |
| Run-time owner tag for one-thread objects | skip the owner's accesses to memory only one thread touches | unsound by construction: a skipped access leaves no record for a later foreign access to race with |
| N1-ATOMIC reach on SQLite and MySQL | inline test for relaxed atomics | ⚪ SQLite's atomics are 0.33 % of cycles; on MySQL 74 % of atomic entries are seq_cst, reach ≤ 0.65 %, realistically 0.1-0.2 %: no leg |
| DE across calls proven synchronisation-free (nosync census) | a census of the "call between" class on whole-program IR | ⚪ Redis 0.51 % of checks (ceiling 0.94), SQLite 0.00, MySQL ≤ 2.08 %: below the 2 % gate, parked 5 Oct |
| DE-5, DE-6, DE-7, DE-8 | finer rules for when a call or a cycle breaks a cover | ⚪ −0.8…+1.4 % |
| IPA-DE | a check in a callee covers one in its caller | ⚪ ≤ 1 % of checks |
| VWIDE, VWIDE-loops alone | run-time verified removal at sites no check dominates | ⚪ −0.9…+1.3 % |
| FE-hot, hot-list N1 (PGO-PLACE), N1-S, N1-LOOPS-∞ | FE-INL or N1 only at hot sites, by profile or statically | 🔴 none beats the full version on the record builds; on MySQL's Debug build FE-hot read 0.922 against FE-INL's 0.855 over stock on Intel (both losses) and 1.080 against 1.100 on AMD |
| FE-PM, N1-PM, N1b | out-of-line entries that save fewer registers | 🔴 MySQL −2 %, SQLite −2…−4 % |
| FE-LAZY, FE-INL-CSE | entry recorded only when needed; one thread-state load per function | ⚪ ±1 % |
| N1-CSE, N1-ATOMIC, LIBCALL-INLINE, N2 | compact, atomic, libc-call and batched variants of the check | ⚪ ±1 % (N2 up to −2 %; N2 as built loses races, since it stores records before the members run, and its exact form saves nothing); N1-ATOMIC on Redis +1.1 % on Intel and +1.6 % on AMD (0.994-1.046 over 4 offsets), not resolved |
| SUBS, SUBS-SEL | a covering record of the same thread counts as a hit; its selective form | 🔴 −2…−18 %; the selective form is unsound |
| Whole-program summaries alone (STC-SUM), STC-TS, allowlists (STC-WL, libfacts) | closed-world facts for STC and SWMR | ⚪ < 1 % of checks |
| LTO, Attributor, extra LLVM passes (XPASS, XP2) | more optimization before instrumentation | ⚪ LTO adds nothing to our analyses; the Attributor miscompiles; the passes undo most of the analyses' check removal on Redis |
| CLONE-ESC, EA-SLOT (EA-SLOT-CONTENTS), ICALL-A2, returns-fresh (named allocators), field chase | finer escape analysis | ⚪ each < 2 % of checks; CLONE-ESC's timed unsound ceiling (OA0): SQLite ≤ +2.6…+4.3 %, memcached ≤ +3.5 %, Redis ≤ +6.0 %, FFmpeg ≤ +10.4 % |
| Whole-program mode for MySQL (MYSQL-WP); top-down parameter facts across units (TOPDOWN-FACTS) | summaries for MySQL's 1,892 units; "this argument is local in every caller" passed to the callee's unit | ⚪ MySQL 0.14 % of checks; the cross-unit facts ≤ 0.14 % on every app |
| Per-field and heap SWMR (SWMR-G, PUBLISH-ONCE), thread roles, thread ids by creation history (THREAD-IDS) | finer may-happen-in-parallel facts | ⚪ each < 3 % of checks |
| NOALIAS, custom lock wrappers (CUSTOM-SYNC), MySQL sysvars, InnoDB latches (INNODB-LATCH), ODR trust (ODR-TRUST) | language and library facts | ⚪ each < 3.2 % of checks |
| RT-SYNC, SLOT-CHURN (SID-ALIAS), RANGE-OVERWRITE | runtime-only changes | ⚪ no effect, or below their gate |
| LO-F, LO-W, LO-B1/B2, sound loop ranges, DE-1, DE-2, DE-4, DE-9 | further lock-ownership and DE rules | ⚪ ≤ 1.4 % of checks, or unsound |

## Table 4. Open and parked

| item | what it is | app | status |
|---|---|---|---|
| **MySQL on the camera-ready compiler** | the camera-ready root's MySQL tree (zyb) against the measured one (cyb), with upstream in the same leg, 4 offsets | MySQL | **resolve first.** Intel 0.998 (A/A 0.995-1.012); **AMD 0.981 at every offset (0.978-0.984)**, A/A 0.990 (within 1 % at three offsets, 0.964 at 00). The objects differ only in TLS names, so the cost is either the TLS change or the runtime (the camera-ready runtime is the quiet-mode runtime, whose hit path gained a test and a branch with the guard off). Next: zyb's objects relinked with the measured tree's runtime, against zyb |
| Camera-ready legs | each app's configuration of record on the one camera-ready compiler, over upstream TSan in the same leg, 4 offsets, with a native arm | all | Redis (qxb/qxn/qxr/qxs) and MySQL (zyb/zys) trees built and IR-identical to the measured trees; memcached, SQLite and FFmpeg trees not built yet; offsets made only for Redis's and MySQL's best; held for the go |
| EVCONF-FIELDS | memcached's guard applied to the owner's reads of its connection's own fields (writes stay checked) | memcached | `a` +3.6 %, `f` +2.9 % on top of table 1b's configuration (tree cwf: 1.114 / 1.088 over EVCONF-ARGS, against cwn's 1.075 / 1.057; A/A 0.993 / 1.001); unsound ceiling of every connection-field read +3.6 % over EVCONF-ARGS (Intel). Audits A58, A58b: sound with conditions under the wider P-X86-FD. Preservation: 4 sites of stock's fd-reuse reports (the wider premise's cost), 3 with the same instrumentation as stock. Awaits the ruling |
| QUIET-FE | function entry and exit not recorded while Redis's main thread is in a skip interval | Redis | FE is 10.3 % of the main thread's cycles; exact only if the runtime rebuilds the thread's stack before its next recorded access; not measured |
| Quiet mode without annotations (AUTO-BIO) | the guard with no annotation: the background threads' start must be proven, not attested | Redis | the automatic variant reads 0.946-0.962 of the best (bio's start is unattested and Redis has no global reset in a run); needs a "written only before threads start" fact; 6-7 days, after the deadline |
| DE-AV (SAME-PTR, DE-10) | DE's covers checked at run time where no dominance holds (a flag set by the first check) or where only the address equality is unproven | SQLite, Redis | unsound ceilings: SQLite availability +8-11 % (AMD), +5.3 % (Intel), address question unresolved; Redis nil. **Parked 4 Oct:** a run-time flag reaches 3-17 % of the executions at a test cost of 0.2-0.3 checks, so the expected gain is about −2…+4.5 % on AMD |
| LO-OBJ-G on the connection's objects | SQLite's VDBE under construction treated as owned by `db->mutex` | SQLite | unsound ceiling +5.8 % on AMD (A/A 0.938-1.030); needs two lock types, a second owner slot and a ruling on how the lock is asserted; parked |
| EA call-site census (X2, OWN-STACK-ENTRY, MAAP-style flags) | per call site, how many pointer-parameter checks a per-site flag or an own-stack entry test would remove | all, MySQL first | MySQL's counting run done: 80.4 G checks in the timed phase, 34.9 % on the accessing thread's own stack; the join with the IR is pending |
| EA-P1's out-parameter cost | a per-argument "stores a pointer" summary bit would restore the top-down fast path | all | not measured |
| Removal-mode DE on FFmpeg | as in table 3 | FFmpeg | +10.2 % over N1 + DynSTC-RT in a screening (mjpeg +29 %); admissible since P-REPORT (3 Oct); not pursued, since FFmpeg's configuration is frozen (3 Oct) |
| MySQL on Intel | FE-INL on the Intel host | MySQL | re-checked 2 Oct with server and client on disjoint CPUs and 24 connections, 4 offsets, A/A 0.994-1.000: FE-INL −3.0 %, the analyses alone −0.2 %. The earlier −6…−12 % came from 36 connections on a 48-CPU set. On the five other scripts only write_only loses (−5.5 %); reads are neutral |
| Stock control | each compiler's stock arm against upstream TSan with the same checks (fork point plus upstream's capture fix) | all | Intel: upstream is faster by 2.5 % on FFmpeg and 2.6 % on memcached, so those older figures are re-based; Redis 1.5 %, not resolved. AMD: memcached 2.7 %, re-based; MySQL the other way. The direct legs above replace the derived figures where they exist |

Not adopted, parked earlier, variants and the runtime-only track: table 6.

## Table 5. Where the analyses are conservative (estimates)

How much each analysis leaves on the table because it cannot prove an alias or ownership fact that holds at run
time. Shares are of executed checks unless marked.

| analysis | what it cannot prove | SQLite | memcached | Redis | MySQL | FFmpeg |
|---|---|---|---|---|---|---|
| DE, dominance | two checks of one invocation hit the same address with no acquire between them at run time and the earlier dominates the later, yet both stay checked (share of all executed checks, 3 Oct, record workloads, P1-v3) | 3.35 % | 0.04 % | 3.44 % | 5.45 % | 5.92 % |
| of which: a call on some path between them | DE refuses unless the callee is proven free of synchronisation; a call to an external or indirect callee / to a local or inlined one | 1.67 % / 0.69 % | 0.04 % / 0 | 3.38 % / 0.00 % | 2.78 % / 1.06 % | 3.53 % / 0.57 % |
| of which: the address question | no call between, only the equality of the two addresses is unproven (array elements with equal indexes, two loads of one field, a loop phi) | 1.0-1.7 % | 0 | 0.06 % | 1.6-2.7 % | 1.8-2.4 % |
| DE, post-dominance | the same, the later check post-dominating the earlier one, a call between them breaking it / ceiling with calls allowed | 0.28 % / 0.37 % | 0.00 % / 0.00 % | 0.30 % / 0.92 % | 0.02 % / 0.53 % | 0.21 % / 0.53 % |
| DE, availability | the same pairs where neither check dominates or post-dominates the other (a flag set by the first check would be needed) | 4.36 % | 0.08 % | 5.79 % | 1.00 % | 8.56 % |
| DE, "checked on every path" | a cover on every path, none dominating (built, audited) | 0.22 % | 0.00 % | 0.82 % | 0.07 % | 0.5 % |
| DE, cycle cut | a cover lost to a path around a loop (built) | 0.92 % | 0.71 % | 0.14 % | 0.08 % | 0.89 % |
| DE, stronger alias analysis (DE-AA) | must-alias from SCEV or points-to analyses | 0 | 0 | 0 | — | 0 |
| EA, all (EA-TL) | checks on memory that only one thread touches or that is consistently ordered in the run, and that EA keeps | 72 % | 55 % | 63 % | 36 % never shared, 13 % written before readers | 70 % |
| EA, pointer parameter | the object is reached through a pointer argument, so the callee cannot tell it is local (share of the row above; MySQL: of all checks) | 53-75 % | 53-75 % | 53-75 % | 76 % | 53-75 % |
| EA, cross-unit parameter facts | "this argument is local in every caller", passed to the callee's unit: the upper bound of what it removes | 0.04 % | 0.05 % | 0.00 % | 0.14 % | 0.01 % |
| EA, own stack | the address lies in the accessing thread's own stack and no other thread touches it | 4.9 % | 5.9 % | 12.5 % | 14.8-24 % | 5.8 % |

- The gap between the EA rows is the point: most of the single-thread mass is real at run time but not provable
  statically, because the objects are heap memory reachable from shared structures or passed through callers that
  also pass shared objects (EA-HEAP: the one-thread heap is 78.7 / 7.5 / 34.0 / 41.9 % of the checks of SQLite /
  FFmpeg / memcached / Redis, and the part freed in the allocating call ≤ 0.6 %). A run-time own-stack test recovers +10 % on MySQL when it skips 34 % of the checks, but
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
  by type-based alias metadata, so S and A are a heuristic split. Lost races (3 Oct, stock 5 runs, each arm 2): SQLite
  loses none in any arm; on memcached A and exact DE lose none, S and S + A + Z lose 2 sites stock reports in every run
  (`clock_handler` reads `stats_state.curr_items` without the stats lock, while `do_item_link` and `do_item_unlink`
  write `curr_bytes` and then `curr_items` under it: under S the `curr_bytes` write covers the `curr_items` write,
  so a genuine race goes unreported). Speed over exact DE, removal mode, Redis on Intel (3 Oct, A/A 0.989-1.009):
  S +8.4 %, A +5.9 %, S + A +11.5 %, S + A + Z +10.6 %; SQLite on AMD (A/A 0.997-1.049): S +11.9 %, A +12.8 %,
  S + A +27.3 %, S + A + Z +31.2 % (create_drop_index_1 +45.5 %, stress2 +18.4 %); memcached on Intel (A/A
  0.997-1.010): S +1.7 %, A +3.9 %, S + A +6.6 %, S + A + Z +6.8 %; MySQL on AMD (A/A 0.988): S +4.1 %, A +1.1 %,
  S + A +6.0 %, S + A + Z +7.9 %. The shadow-proxy rule of RedCard, the core of S that keeps "at least one race is
  reported where stock reports one", was counted and not adopted, since it reports a different race than stock: it
  would remove only memcached 0.13 %, Redis 1.06 %, SQLite 0.77 %, FFmpeg 0.11 % of executed checks, against S's
  9.5-22.5 %, because most fields are touched by a memory intrinsic or lack one proxy that accompanies every access.

## Table 6. Every other optimization tried

Variants of the levers above, optimizations whose result is a count or a closure rather than a resolved timing, and
the runtime-only track (parked from the paper on 25 and 28 Sep). Shares are of executed checks unless marked.

| optimization (aliases) | what it is | result | standing |
|---|---|---|---|
| **DE** | | | |
| DE package (tier-A T4, T4+E) with DE-B selective peeling | DE-2 merge + DE-3 loop ranges + DE-B, with the inexact DE of the time and with exact DE (+E) | over the shipped compiler: T4 FFmpeg +12 % (mjpeg +39 %), SQLite +0.2, memcached +0.8, Redis +1.4, MySQL −3.5 %; T4+E SQLite +0.8, memcached +3.6 (Intel), Redis −2.6, MySQL +3…+6, FFmpeg +0.5 %; DE-B reaches 0-0.22 % | superseded by exact DE and the levers of tables 2-3 |
| N3, N4, N8 | DE path flags; covers across loop iterations; trimmed entry/exit | ≤ 1 % each | closed |
| DE-8b | C allocation functions treated as synchronisation-free | 0 % on every app, and unsound (stock 10/10, flag 0/10) | dropped |
| relaxed-atomic rule (`-tsan-de-relaxed-atomic-nosync`) | relaxed atomics do not break a cover (premise A8) | raised SQLite's loop-guard skips from 0.74 % to ≈ 9.2 % of checks | on in every build; A8 pending |
| pure-asm rule | inline asm treated as synchronisation-free | 0 on SQLite and memcached, ≈ 0 on FFmpeg | closed |
| anticipation, acquire tolerance, available-checks dataflow (census of 1 Oct) | a cover moved up to where the check is anticipated; covers across an acquire; must-available checks | anticipation subsumed by verified placement (census mass Redis 16 %, memcached 10 %); acquire tolerance 0.00 %; dataflow SQLite 0.56, FFmpeg 0.48, memcached 0.04, Redis 0.02 % | closed |
| POSTDOM-TERM | post-dominance with proven termination (ASYNC-TERM) | memcached 0.03, SQLite 0.05, Redis 0.27 % | in the tree |
| **EA** | | | |
| U2 | unsound bound: named arguments never escape (the own-stack bound) | over T1: SQLite 0.996, memcached 0.972, Redis 0.963, MySQL 0.981, though it removes 6.7-13.2 % of sites | closed: no headroom |
| integer-copy closure off (premise A7) | EA without following pointers copied as integers | over shipped: SQLite −0.2, memcached +0.9, Redis +1.0, MySQL −2.7 %; on FFmpeg the closure keeps 0.86-1.01 pp of checks | closure kept on; A7 not adopted |
| EA-7; the record-compare chain | arguments of external or address-taken functions; three chained refinements | EA-7 +5.5 k static sites; unsound bound (`-tsan-ea-assume-nonescaping-args`) ≈ 5.6 % of SQLite's checks | shelved 25 Sep |
| EA-WP | EA with whole-program summaries | 0 % of the one-thread heap mass on three apps | closed |
| **STC, SWMR, LO** | | | |
| STC-1, STC-2, STC-3, STC-4, join-aware STC, STC-CALLEE, STC-STRONG (i) (O-STC), DYN-1 | finer single-threaded-context rules; dropping DynSTC's guards | STC-1: 0 of 32.1 G checks on SQLite, 19 of 20.9 G on memcached; single-threaded executions on SQLite 0.09 %; join-aware cap 2.44 %; DYN-1 0.08 % of guard runs | closed |
| MAIN-ONLY (STC-STRONG (ii)) | prove statically that Redis's reads happen only on the main thread | statically impossible; dynamic ceiling 22.0 % of Redis's checks; unsound "SWMR-ok skip" oracle 1.075 | closed 29 Sep; the quiet threads take this mass at run time |
| DYN-2 | where DynSTC's guard skips, by counters | FFmpeg's stream copy 100 % skipped (single-threaded), which is the whole gain; elsewhere 97.6-99.98 % of guarded checks still run; Redis's background threads are never joined, so the guard cannot skip there | closed; DynSTC-RT is the line |
| LO-unwind | an invoke's unwind edge inherits the callee's acquisitions | refuted: the race is reported 10/10 with LO | closed |
| SQLITE-CW | closed world for SQLite (resolving `xMutexEnter`) | 0.02 % of executed accesses | closed |
| LO-C | POSIX calls transparent to LO | sound, no gain | closed |
| SWMR-H | a location written only before its publication is unchecked afterwards | U1 ceiling: memcached +0.2 %, Redis 0, SQLite 0; the audit found 14 lost-race paths | parked 24 Sep |
| SWMR-ROOTS on SQLite, MySQL, Redis | memcached's rule on the other apps | T2 globals: 1.7 / 2.7 / 0.2 % of checks; ≤ 0.5-1 % expected | not built |
| G-EA | escape analysis under LO-OBJ-G's guard for SQLite's private b-trees | fires on 99.9 % of guarded checks but loses: stable-4 1.071 → 1.029 (AMD, 30 Sep) | closed 30 Sep |
| LO-OBJ-G spec completion | BtShared fields spec v7 does not name | ≈ 1.2 % of create_drop_index_1's checks | not built |
| LO-OBJ-G Pager/Wal roots | Pager and Wal objects under their b-tree's mutex | Wal 10.5 % + Pager 2.2 % of SQLite's locked mass | declined with P-PAGER |
| SQLITE-KEY | the comparator's key and payload as spec extensions; the unpacked record on the stack | ≈ 1.5 % + ≈ 1.6 % of checks | parked 2 Oct |
| SQLite sharable guard | a per-object guard for SQLite's shared cache | cannot reach the shared-cache mass | closed |
| PERELEM | per-element locksets: a struct guarded by its own mutex | memcached ≈ 2.3 % (provable part ≈ 0), the others ≈ 0 | parked |
| TLS-rooted escape analysis (LO-TID) | objects reachable only from a `thread_local` root are thread-private | MySQL ≈ 7 % of checks on Debug, ≈ 0.8 % on Release; other apps 0; audits A36, A36b: sound with conditions | parked 1 Oct (camera-ready MySQL is Release) |
| **N1, FE and other inline paths** | | | |
| N1-ST =miss | N1-ST's flag read only on a miss | FFmpeg +9.8 % (front +16.9 %), Redis +2.1, SQLite AMD +2.0 % (27 Sep); FFmpeg −6.6 % against front (1 Oct) | front kept |
| N1-ST (b), (c) =fs | the flag hoisted; the flag kept in the fast state's bit 30 | (c) equals front on FFmpeg (0.994), unresolved elsewhere; every N1-ST variant on SQLite −3.4…−4.4 % over stock; (b) not built | front kept |
| N1-ST-WORKER | N1-ST's front test dropped in worker-only code | 0.00 % of executed checks | closed 3 Oct |
| N1-SPLIT | N1's miss blocks moved to cold sections | 0 of 59,808 miss calls outlined | closed |
| N1-PAIR | one 32-byte load tests two cells | subsumed by DE-2R | closed |
| N1-CSE across calls | sibling shadow tests shared across calls and blocks | 25-31 % of executed sites share a sibling; ≤ 1 % expected | not built |
| FE-INL exit-max=1; unified exit | FE-INL variants | exit-max=1: MySQL Intel −5.2 %; the unified exit is byte-identical to FE-INL | dropped |
| FE oracle; TSan-aware inlining | function entry and exit removed (unsound ceiling); inlining hot small callees before instrumentation | ceiling Redis +23.5 pp, MySQL +13.6, SQLite +4.4; FE events by callee not counted | ceiling; FE-INL is the sound part |
| call-cost ceiling (N1/N2-real) | the share of run time spent in the runtime calls of checks that hit | SQLite 21.0 %, FFmpeg 17.5 %, Redis 25.0 %, memcached ≈ 0 (24 Sep) | bounded N1 and N2 (table 2-3) |
| FE-NOFRAME | no entry/exit in functions without accesses | — | not taken: changes report stacks |
| MEMINTR-INLINE | small constant-size memcpy/memset inlined | ≈ 0.5 % memcached, ≤ 1 % Redis expected (intrinsics ≤ 32 bytes are 4.95 % of Redis's main thread) | not recommended |
| **Annotation-based and ownership variants** | | | |
| EVCONF-INTERCEPT write side (X5) | the guard on sendmsg/writev iovecs | unsound ceiling 1.007, inside the A/A (Intel V4, 5 Oct) | closed 5 Oct |
| fresh item until linked | memcached's new item: its copy and key hash before it is linked | ≤ 1.1 % (estimate) | not built |
| one-sided `_nosrc` copies under EVCONF | covered source, uncovered destination | ≤ 0.8 % (estimate) | not built |
| quiet-thread refinements | the skip bit folded into the fast-state load; re-read only after calls that can synchronise | the code that never skips costs +6.4 % of M's cycles; re-read ≤ ≈ 0.3 %; the fold helps AUTO only | parked |
| R1-R7 (rejected 4 Oct) | R1 EVCONF claims the whole connection; R2 quiet-mode allocation stacks; R3 the worker's stats mutex skipped; R4 a SQLite connection latch; R5 refcount holders' reads; R6 DOM-FS; R7 EVCONF guard hoist | R1 unsound; R2 changes reports (8.2 % of M); R3 false reports (≈ 6 % of cycles); R4 waits for a ruling; R5 trusts the program's protocol; R6 = DE-AV's parked form; R7 ≈ 0.3 % | rejected |
| OWN-HANDOFF | a buffer's checks skipped while the program's hand-off protocol says one thread holds it | Redis +13.9 % over P1-v3; unsound hand-off oracle 1.277; memcached's genuine hand-off ≈ 1.3 % of checks | not adopted: trusts the program's protocol; the quiet threads replace it |
| OWN-CONN | an owner guard on SQLite's private connections | reach 21.7 % of stable-4's checks, ≈ +21 % estimated | withdrawn 29 Sep (premise P-CONN not asserted in code) |
| OWN-STACK | checks skipped on the accessing thread's own stack | unsound ceiling +5.3 % net on MySQL; sound form ≈ −1 % | parked 2 Oct |
| SPIN-ACQ | a loop of atomic loads polls relaxed and acquires only on a change | Redis ≤ +14 % (ceiling) | parked 28 Sep |
| FFmpeg heap hand-off | buffers handed between FFmpeg's threads | rests on a program-protocol premise | closed 1 Oct |
| **Runtime-only track** | | | |
| DD-EXACT (DD-COST; A = DD-CHEAP, B1 = DD-BIG) | TSan's deadlock detector: A an exact lock-order table, B1 a larger table | A is sound: memcached 0.998 and 0.988; MySQL 1.041 in one leg, 0.996 over stock in a later one. B1: memcached +19.4…+21.7 %, but it drops lock-order reports (never race reports). Detector off: memcached ×1.29 | parked 1 Oct |
| RT-SYNC-LF | lock-free acquire loads | fires on 99.990 % of Redis's acquire loads; 1.025 against an A/A of 1.031 | not resolved; parked |
| RT-RANGE (R1 SIMD, R2) | faster range checks | R1 ≈ 2-4 % (estimate) | parked 25 Sep |
| RANGE-HIT-SKIP | a range whose cells all hit is skipped | all-hit share FFmpeg 81, SQLite 73, Redis 50, memcached 11 %; bound ≤ 0.53 % | closed 28 Sep |
| RANGE-VEC | vectorised walk of hit cells | bound ≤ 0.9 % (SQLite), < 0.4 % elsewhere | not opened |
| RANGE-UNIFORM | on a range miss over identical cells, one race check and a wide store | ≈ 2 % bound; a microbenchmark saves 23 % per cell against a 30 % bar | closed 29 Sep |
| RT-ALLOC | faster freed/reset shadow fill; cached allocation stacks | Redis ≈ 11 % of cycles (mass) | parked 25 Sep |
| RT-SLOT, SLOT-PREF | slot preemption | 1.5 % of preemptions find a free slot; slot chains ≈ 1.7 % of memcached's user time | closed |
| empty-epoch elision, early reset | fewer global resets | 12.2 % of memcached's epochs are empty, ≈ 12 % fewer resets; early reset ×1.26 resets | not proposed |
| ALLOC-COVER, RANGE-COVER | allocation and range records that cover later checks | proxy 5.6 % (Redis), 7.0 % (FFmpeg) | out of scope by decision |

No result exists yet for: sendmsg batching, the interceptor bypass, EA-P1's out-parameter cost (table 4), and the
small EA precision counts (atomics on EA-local objects, MySQL vptr loads, copies within one object).

## Notes

- **Method.** Each arm is built at four code offsets (0/16/32/48 bytes mod 64) and scored by the mean of per-offset
  ratios; each leg has an A/A arm, and a result counts only if it is above the A/A range at every offset. Screenings
  use two offsets. On memcached the CPU layout and the workload shape change the size of an effect, so each figure
  names its layout; AMD memcached legs before 4 Oct 18:40 that ran the V3 shape (pipeline 16, no multi-key get) are
  labelled V3. SQLite runs on NVMe at N ≥ 6 per cell on AMD: tmpfs neither narrowed the A/A nor kept the ratio.
- **Baseline.** Upstream TSan is the fork point plus upstream's fix for unchecked fields of escaped locals
  (tsan-pristine-up59b2-0f1aed148576). Older figures over our stock arm (each project compiler with every pass off)
  are re-based by the stock-control ratio where upstream was faster beyond the A/A, and marked "derived".
- **memcached's other workloads, AMD, directly over upstream TSan** (disjoint CPUs, 2 offsets, the configuration of
  2 Oct): default input +2.8 %, pipelining +3.5 %, 32-key gets +6.3 %, long keys +16.8 %, the chosen mix +30.4 %.
- **The runtime's cost on the stock path.** With the same application objects, the project's runtime executed 1-3.5 %
  more instructions than upstream's (FFmpeg copy +1.0 %, mjpeg +3.5 %, memcached +2.9 %), on the miss and eviction
  paths of the access entries. The cleared runtime (A44) matches upstream's instruction count; it reaches upstream's
  time on memcached but recovers only about a fifth of FFmpeg's 2.7 % gap, a code-placement effect. 64-byte alignment
  trades FFmpeg +1.3 % against memcached −1 %, so upstream stays the base.
- **TLS access (fixed 5 Oct).** Exporting the runtime's thread-state variable made every access entry load its TLS
  offset from the GOT: the same instructions, +3.1 % cycles and −2.4 % requests per second on Redis's stock path
  (Intel). The camera-ready runtime keeps the storage hidden and exports an alias. Timed effect on MySQL: table 4.
- **The camera-ready compiler** `tsan-cr-8345a0396fa5` (frozen 5 Oct 09:18): the cleared integration compiler (audits
  A42, A44) + Redis's quiet mode (A43, A45-A56) + its whole-program proof (A57-A57c) + the TLS fix + a wake-up fix
  + the guard's exports + memcached's EVCONF line through INTERCEPT and IN-BOUNDS (A46-A58b). Gates at freeze: IR
  suites 191, quiet-mode tests 51/51, check-tsan in 12 configurations, go-check. EVCONF-FIELDS is not in it.
- **Race preservation.** Every row of tables 1-2 loses no race stock TSan reports under the premises in
  `soundness-fixes.md` §6, checked by IR tests, check-tsan, reproducers with controls and an independent audit.
  Application runs against stock, 10 runs per arm, races matched by location pair:
  - 3 Oct, every best configuration and best + DE all-paths and cycle cut: memcached 4 races kept (9 more appear in
    only 1 of 10 stock runs), SQLite 3 kept, MySQL 232 and 236 kept over 5 sysbench scripts, Redis (no race on its
    benchmark) all 10 seeded races kept, three of them placed where the DE covers fire. FFmpeg's stock reports no
    race, so it certifies nothing.
  - memcached's final configuration (cwn, 5 Oct): 4 kept, 0 lost; 7 sites stock reports only sometimes, each with
    the same instrumentation as stock at its lines. EVCONF-FIELDS (cwf): 4 sites of stock's fd-reuse reports lost,
    the wider P-X86-FD's cost.
  - Redis's quiet mode (final root, ranges on and off), 10 runs × 11 seeded scenarios: no seeded race lost; plain
    record commands: 0 races in every arm, 1.23 G range checks skipped.
  - A site is classified as kept, lost to eviction (stock reports it in fewer than 10 of 10 runs, Fisher p > 0.05,
    ≥ 3 threads on the granule), the same instrumentation as stock (same calls per line, at least stock's count, no
    guard), a premise's pair class, or lost; the worst class of a site counts.
- **Premises awaiting a ruling** (§6 of `soundness-fixes.md`): P-X86-FD (narrow form: EVCONF; wider form:
  EVCONF-FIELDS), EVCONF's libevent callback contract, MALLOC-ATTR (the quiet mode's fresh-call rule, which Redis does
  not use), and the provisional A5, A8 and A9.
- **MySQL server deaths (closed 4 Oct).** 2 in 104 runs of arms with N1, 0 in 342 others, on 27-28 Sep Debug roots: one
  lost connection, one InnoDB debug assertion. Neither root is an ancestor of today's N1 code. A sweep of every N1 run
  since (Redis 715, FFmpeg 660, memcached 258, SQLite 184, MySQL 148) finds no other N1 failure; MySQL since then:
  0 deaths in 670 runs.
- **Compile time** of the paper's analyses over stock (CPU): SQLite +20 %, memcached +8 %, Redis +9 %, FFmpeg +16 %.
