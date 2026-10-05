# TSan instrumentation optimizations: results

State: 5 Oct 2026, 12:10. Speedups are over upstream TSan measured in the same leg ("direct") unless a cell says
otherwise. `a` = AMD (2 × EPYC 9115), `f` = Intel (Xeon w9-3495X). Method, baseline and race preservation: Notes.
Longer earlier versions are in the history (commits 3abcd2c, ec0d58e).

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
| Redis | FE-INL + N1 + quiet-thread phase guard, annotated, range skip on (tree qpr; camera-ready qxr is IR-identical) | 🟢 **+38.7 %** (1.349-1.452, 4 offsets, A/A 0.980-1.003); +34.1 % in an earlier leg on the final root | 🟢 **+31.4 %** (4 offsets, A/A 0.987-1.033) | audits A43, A45-A57c; preservation: no seeded race lost; premises FIELD-ADDR, A12-DATA, P-DICT adopted |
| SQLite | LO-OBJ-G | 🟢 **+23.5 %** with spec v6 (1.180-1.302, 4 offsets × N=4; stress2 +23.9 %, create_drop_index_1 +23.1 %; +18.5 % over our stock arm, the faster base here); spec v7 (with the latch) +20.0 % and +20.9 % in two legs | — | v7 audited; the camera-ready leg carries v6 beside v7 |

- **memcached's levers, V4, AMD / Intel:** EVCONF-RANGES +18.5 / +13.9 %, EVCONF-ARGS +7.3 / +7.2 %, EVCONF-INTERCEPT
  +7.5 / +5.7 %. The configuration before them reads +31.1 / +23.3 % over upstream in the same direct legs, and their
  product matches the direct ratio (AMD 1.367 against 1.375, Intel 1.290 against 1.286). EVCONF-FIELDS would add
  +3.6 / +2.9 % (table 4).
- **Redis's quiet mode over FE-INL + N1 in the same leg:** AMD +26.3 % (earlier leg +29.6 %), Intel +25.6 % (other
  legs +27.4 %, +31.4 %); the range skip's own share is +3.6…+9.9 %. The unsound ceiling of skipping all of the main
  thread's plain checks is +36.5 % (Intel).
- **One configuration for every app** (derived): N1 + N1-ST + DynSTC-RT gives FFmpeg +24 %, Redis +6.5 %, and loses on
  SQLite (−0.5…−3.6 %), memcached (−6.7 %) and MySQL (−2.6 %).
- **The paper's analyses alone** (P1-v3, below) are not separable from stock on any app (0.98-1.02).
- **Ceilings for removing checks** (profile oracles: memory touched by one thread / also Eraser-consistent): SQLite
  +16 / +53 %, memcached +6 / +11 %, Redis +17 / +27 %, MySQL −1 / +3 %, FFmpeg +31 / +41 %. They come from 26 Sep
  profiles of older workloads and count memcached's request keys and connection fields as shared, which is why the
  annotations exceed them.

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
| **FE-INL + VWIDE-loops** | plus run-time verified check removal inside loops | `a` ⚪ +0.1 | `a` ⚪ +0.6 | `f` 🟢 **+7.2** | `a` 🟢 **+4.6** | `f` 🟡 +1.8 |
| **FE-SINK** | the function-entry call is moved to the first point that needs the frame | `a` 🟡 +1.1 | `a` 🟡 −1.7 | `f` 🟢 **+4.1** | `a` 🟢 **+2.8** | `f` 🟡 −1.2 |
| **N1-L** | N1 only in loops with at most 20 checks | `a` ⚪ +0.3 | `a` ⚪ −0.9 | `f` 🔴 −2.5 | `a` ⚪ −0.1 | `f` 🟢 **+3.0** |
| **MEMINTR** | a memcpy from a constant or private source has only its destination checked | `a` 🟡 +0.7…+2.5 | `a` ⚪ +0.9 | `f` 🔴 −2.5 | `a` ⚪ +0.3 | `f` ⚪ 0.0 |
| **EA-CONTENTS** | a pointer read from a container no longer makes the container shared | — | `a` 🟢 **+1.4** (on against off) | — | — | — |
| **SWMR-ROOTS** | no checks on reads of a global whose only write precedes every reader thread | — | `a` 🟢 **+0.9** on top of EVCONF | — | — | — |
| **EVCONF** (annotation) | objects annotated as owned by one thread are unchecked while a run-time guard holds (no idle-timeout thread, no external storage, connection never lent) | — | `a` 🟢 **+22.7** over stock on shared CPUs; +32.2 with the two rows above on disjoint CPUs | — | — | — |
| **EVCONF-RANGES** (annotation) | EVCONF's guard applied to memset/memcpy on an owned object (constant length in its type, or any length under IN-BOUNDS); one memset zeroing each response object was 16.6 % of the cycles | — | `a` 🟢 **+18.5**, `f` 🟢 **+13.9** (V4; AMD V3 +10.3); Intel equals the unsound ceiling (+14.1) | — | — | — |
| **EVCONF-ARGS** (annotation) | EVCONF's confinement carried into the hash function's key argument: a clone of MurmurHash3 with the key reads unchecked while the guard holds, called only where the key is a covered request key | — | `a` 🟢 **+7.3**, `f` 🟢 **+7.2** (V4; AMD V3 +5.1); unsound ceiling of all key reads +8.2 (Intel) | — | — | — |
| **EVCONF-INTERCEPT** (annotation) | EVCONF's guard around the libc calls that scan the confined read buffer: memchr, strlen and the request-key side of bcmp | — | `a` 🟢 **+7.5**, `f` 🟢 **+5.7** (V4, 4 offsets); unsound ceiling +6.5 (Intel) | — | — | — |
| **Quiet-thread phase guard** (annotation) | Redis's main thread, which runs ≥ 99 % of the checks, skips its checks (range checks included) while every other thread is quiet since a release it acquired; annotations register the threads' park objects and attest their start | — | — | `a` 🟢 **+26.3**, `f` 🟢 **+25.6** | — | — |
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
| Loop guard (T8) and the T10 package | loop guard: a loop-invariant access is checked once per synchronisation-free stretch and re-checked when the shadow generation moves; T10: the older compiler's full lever package (ALL, with N1-L) including it | ⚪ over the paper's analyses, 4 offsets (4 Oct): Redis +2.2 / +3.1 %, memcached 0 / +1.6 %, SQLite not resolvable (A/A ±7 %), MySQL −0.1 / 🔴 −4.2 % |
| Earlier levers on SQLite v7 | N1, N1-L, MEMINTR, FE-INL on top of the configuration of record | ⚪ FE-INL +4.5 %, MEMINTR +3.5 % at the edge of a wide A/A (0.974-1.053), N1 🔴 −18.5 %; FE-INL and FE-INL + MEMINTR are arms of SQLite's camera-ready leg |
| The old "same location" rule (unsound reference) | struct fields cover each other (S), array elements cover each other (A), a narrower check covers a wider one (Z) | 🔴 unsound: S loses a real memcached race. Measured as the related-work bound: Redis +6-12 %, SQLite +12-31 %, memcached +2-7 %, MySQL +1-8 % (table 5 notes) |
| LO-OBJ-ARGS | LO-OBJ-G's lock ownership carried into SQLite's record comparison and the schema-name compares | ⚪ unsound ceiling 1.023 (AMD, 4 offsets × N=4), every offset inside the A/A (0.972-1.063); closed 5 Oct |
| LO-OBJ-RANGES | LO-OBJ-G's guard applied to SQLite's memcmp and VDBE memcpy | ⚪ reaches neither site: the copied registers are read without `db->mutex`, and the page side arrives through a function pointer (4 Oct) |
| EA interceptor toggle in per-unit builds | EA's libc-call toggle without the whole-program definitions list (A4 adopted) | ⚪ SQLite 0.977 (A/A 0.907-0.983), MySQL 1.003 (A/A 0.992-1.001); closed 5 Oct |
| memcached's flag globals | the unlocked reads of `settings`, `expanding` and `hashpower` | unsound ceiling +4.5 % (Intel); no sound route: admin commands write `settings`, and `expanding` is a genuine benign race; closed 4 Oct |
| Quiet mode beyond Redis | the phase guard on the other four apps | ⚪ ≤ 0.4 % of checks fall in quiet intervals (census, 4 Oct) |
| MySQL THD owner reads | a connection's own THD fields read by its thread, under an owner guard | ⚪ ≤ 2.35 % of plain checks, below the 3 % bar (4 Oct) |
| Run-time owner tag for one-thread objects | skip the owner's accesses to memory only one thread touches | unsound by construction: a skipped access leaves no record for a later foreign access to race with |
| N1-ATOMIC reach on SQLite and MySQL | inline test for relaxed atomics | ⚪ SQLite's atomics are 0.33 % of cycles; on MySQL 74 % of atomic entries are seq_cst, reach ≤ 0.65 %, realistically 0.1-0.2 %: no leg |
| DE across calls proven synchronisation-free | a census of the "call between" class on whole-program IR | ⚪ Redis 0.51 % of checks (ceiling 0.94), SQLite 0.00, MySQL ≤ 2.08 %: below the 2 % gate, parked 5 Oct |
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

## Table 4. Open and parked

| item | what it is | app | status |
|---|---|---|---|
| **MySQL on the camera-ready compiler** | the camera-ready root's MySQL tree (zyb) against the measured one (cyb), with upstream in the same leg, 4 offsets | MySQL | **resolve first.** Intel 0.998 (A/A 0.995-1.012); **AMD 0.981 at every offset (0.978-0.984)**, A/A 0.990 (within 1 % at three offsets, 0.964 at 00). The objects differ only in TLS names, so the cost is either the TLS change or the runtime (the camera-ready runtime is the quiet-mode runtime, whose hit path gained a test and a branch with the guard off). Next: zyb's objects relinked with the measured tree's runtime, against zyb |
| Camera-ready legs | each app's configuration of record on the one camera-ready compiler, over upstream TSan in the same leg, 4 offsets, with a native arm | all | Redis (qxb/qxn/qxr/qxs) and MySQL (zyb/zys) trees built and IR-identical to the measured trees; memcached, SQLite and FFmpeg trees not built yet; offsets made only for Redis's and MySQL's best; held for the go |
| EVCONF-FIELDS | memcached's guard applied to the owner's reads of its connection's own fields (writes stay checked) | memcached | `a` +3.6 %, `f` +2.9 % on top of table 1b's configuration (tree cwf: 1.114 / 1.088 over EVCONF-ARGS, against cwn's 1.075 / 1.057; A/A 0.993 / 1.001); unsound ceiling of every connection-field read +3.6 % over EVCONF-ARGS (Intel). Audits A58, A58b: sound with conditions under the wider P-X86-FD. Preservation: 4 sites of stock's fd-reuse reports (the wider premise's cost), 3 with the same instrumentation as stock. Awaits the ruling |
| QUIET-FE | function entry and exit not recorded while Redis's main thread is in a skip interval | Redis | FE is 10.3 % of the main thread's cycles; exact only if the runtime rebuilds the thread's stack before its next recorded access; not measured |
| Quiet mode without annotations (AUTO-BIO) | the guard with no annotation: the background threads' start must be proven, not attested | Redis | the automatic variant reads 0.946-0.962 of the best (bio's start is unattested and Redis has no global reset in a run); needs a "written only before threads start" fact; 6-7 days, after the deadline |
| DE-AV | DE's covers checked at run time where no dominance holds (a flag set by the first check) or where only the address equality is unproven | SQLite, Redis | unsound ceilings: SQLite availability +8-11 % (AMD), +5.3 % (Intel), address question unresolved; Redis nil. **Parked 4 Oct:** a run-time flag reaches 3-17 % of the executions at a test cost of 0.2-0.3 checks, so the expected gain is about −2…+4.5 % on AMD |
| LO-OBJ-G on the connection's objects | SQLite's VDBE under construction treated as owned by `db->mutex` | SQLite | unsound ceiling +5.8 % on AMD (A/A 0.938-1.030); needs two lock types, a second owner slot and a ruling on how the lock is asserted; parked |
| EA call-site census (X2, OWN-STACK-ENTRY) | per call site, how many pointer-parameter checks a per-site flag or an own-stack entry test would remove | all, MySQL first | MySQL's counting run done: 80.4 G checks in the timed phase, 34.9 % on the accessing thread's own stack; the join with the IR is pending |
| EA-P1's out-parameter cost | a per-argument "stores a pointer" summary bit would restore the top-down fast path | all | not measured |
| Removal-mode DE on FFmpeg | as in table 3 | FFmpeg | +10.2 % over N1 + DynSTC-RT in a screening (mjpeg +29 %); admissible since P-REPORT (3 Oct); not pursued, since FFmpeg's configuration is frozen (3 Oct) |
| MySQL on Intel | FE-INL on the Intel host | MySQL | re-checked 2 Oct with server and client on disjoint CPUs and 24 connections, 4 offsets, A/A 0.994-1.000: FE-INL −3.0 %, the analyses alone −0.2 %. The earlier −6…−12 % came from 36 connections on a 48-CPU set. On the five other scripts only write_only loses (−5.5 %); reads are neutral |
| Stock control | each compiler's stock arm against upstream TSan with the same checks (fork point plus upstream's capture fix) | all | Intel: upstream is faster by 2.5 % on FFmpeg and 2.6 % on memcached, so those older figures are re-based; Redis 1.5 %, not resolved. AMD: memcached 2.7 %, re-based; MySQL the other way. The direct legs above replace the derived figures where they exist |

Not adopted or parked earlier: OWN-HANDOFF (Redis +13.9 % over P1-v3, but it trusts the program's own thread protocol),
OWN-CONN, DD-EXACT (deadlock detector table: memcached +19.4 %, unsound as committed), TLS-rooted escape analysis
(Debug build only), per-element locksets (1.7 % of memcached's checks), SPIN-ACQ, SWMR-H, OWN-STACK (sound form ≈ −1 %).

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
