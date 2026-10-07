# TSan instrumentation optimizations: results

State: 7 Oct 2026. Earlier, longer versions are in the history: f02509d (last before this layout), b8d5c56, 3abcd2c.
Soundness fixes and premises: `soundness-fixes.md`. Where the overhead goes: `hotspots.md`.

## Terms

- **Speedup**: stock TSan's run time over the optimized build's, in the same leg, geometric mean over 4 code offsets
  (0/16/32/48 bytes mod 64). The base is stock TSan built by the same compiler and runtime; a figure marked ᵘ is over
  upstream TSan (the LLVM fork point plus upstream's fix for unchecked fields of escaped locals,
  `tsan-pristine-up59b2-0f1aed148576`), where no same-compiler stock arm was timed.
- **Host**: AMD (2 × EPYC 9115) unless marked `f` = Intel (Xeon w9-3495X). `$EXTRA` is the campaign's data area
  on the lab host (`/extra/<user>`).
- **Sound**: loses no race stock TSan reports, under the premises of `soundness-fixes.md` §6. Every figure is the best
  sound result measured; unsound figures appear only as ceilings, marked U.
- **A/A**: the base run twice in the same leg. A gain counts when it lies beyond the A/A range at every offset.
  A/A ranges stay in the notes, never in summary cells.
- **Kind** of an optimization (how its decisions are obtained):
  - **S** static: the compiler alone, no annotation;
  - **R** run-time checked: no annotation; the compiler's decisions are backed by run-time guards or latches that keep
    them sound;
  - **G** generated from the program's own assertions (a spec the generator writes);
  - **A** annotation: a hand-written spec, checked by the compiler; **A+R** when also guarded at run time;
  - **RT** runtime-only change (no compiler decision); **U** unsound (ceiling or reference only).
- **P1-v3**: the paper's analyses EA, LO, STC, SWMR, DE with every soundness fix; a covered check keeps an inline hit
  test (verified removal).
- **The one configuration**: one compiler configuration and one runtime for every app (runtime changes R3, phase-skip
  entries without a forced inline hit test, and R2, a settled thread's hooks go quiet; not the LO-OBJ-G premise R3), root
  `tsan-cc-6815ac749368`, refrozen 7 Oct as `tsan-cc-744024407b56` (reproduces it: Redis rcj4 0.995, memcached mcj4
  0.997, MySQL myj4 1.002, all inside the A/A). It builds with **-wp** (whole-program mode and the phase derivation)
  only where the program's record uses it (§ "The one configuration").
- **Markers**: 🟢 gain beyond the A/A · 🟡 1-2 % or unresolved · ⚪ within ±1 % · 🔴 loss · — not measured.
- Names in parentheses are aliases the records use. Tables 2-6 and the history table hold every optimization with a
  result of any kind (inventory of 5 Oct, about 180 items, plus 6-7 Oct).

## Table 1. Summary (AMD)

| app | workload | submitted paper | (1) no annotations, static | (2) no annotations, run-time checked | (3) spec generated from the program's assertions | (4) our annotations |
|---|---|---|---|---|---|---|
| SQLite | threadtest3: stress2, create_drop_index_1 | 1.71× | 1.02×ᵘ ⚪ (the paper's analyses) | — | **1.16×** (LO-OBJ-G, gen6 = gen8's output; run-time lock guard; inputs named by hand: note 1) | **1.25×** (LO-OBJ-G, hand-written spec v7; run-time guarded) |
| FFmpeg | four transcodes of one film | 1.57× | — | **1.29×**ᵘ (DynSTC-RT + N1 + N1-ST) | — | — |
| Redis | 7 data-heavy commands, 8 I/O threads | 1.45× | 1.10×ᵘ (FE-INL + N1) | **1.54×** (FE-INL + N1 + quiet threads, spec derived from the whole program; the one configuration) | — | 1.40× (hand-written quiet-thread lines; earlier leg; superseded by (2)) |
| MySQL | Release, sysbench insert / update / delete, 24 connections | 1.16× | **1.13×**ᵘ (FE-INL) | — | — | — |
| memcached | pipelined 32-key gets, 190-byte keys (V4) | 1.07× | **1.02×** (the analyses + EA-CONTENTS + SWMR-ROOTS + FE-INL + MEMINTR) | in progress (EVCONF-CHECKED) | — | **1.81×** (EVCONF line of 7 fields, compiler-checked, run-time guarded; the one configuration) |
| Chromium | — | 1.39× | not re-measured | | | |

1. **SQLite, (3) and (4).** The generator reads SQLite's own `sqlite3_mutex_held` assertions (about 85 in btree.c)
   and writes the spec (gen5, then gen6; gen7 and gen8 derive the predicates and the enter/leave functions, and gen8's
   spec compiles to code identical to gen6's). Inputs still named by hand (audit A61f): the lock `BtShared.mutex`,
   `removeFromSharingList`, the allocator names, the parameter name `iDb`, the struct `CellInfo`, the SQLite mutex
   API names and 12 field names. The run-time guard `__tsan_lo_obj_any_held` checks that the lock is held. Leg zg64b
   (A/A 1.016): gen6 1.157 (create_drop_index_1 1.143, stress2 1.166), hand-written v7 1.250, both over stock built
   by the same compiler (tsan-cr-8345a0396fa5). Audits A61d, A61e (gen6), A61f (gen8, sound with conditions);
   preservation GPRES6. gen5 read 1.14× (zg5st4). The static 1.02× is over upstream, inside the A/A; over the
   same-compiler stock arm it read 0.98×.
2. **Redis (2).** The quiet threads' 12 lines are derived by the compiler from the whole program (AUTO-BIO; audits
   A60-A60c), with the same skips as the hand-written lines; the phase guard and its latch check at run time. Leg rcs4
   (A/A 1.005): 1.544 over stock built by the same root and runtime, steady at every offset. Premise P-DICT (the narrowed
   form of CFG-TABLE, adopted 4 Oct; `soundness-fixes.md` §6).
3. **memcached (1).** Leg mcnf4 (A/A 1.002): mcf 1.019, the paper's analyses alone (mcn) 1.016, all-paths DE adds
   nothing (mca 1.018); copy mcnf4a agrees to 0.001. N1 is off: it costs ~5 % here (88 % of checks miss; mcz 0.951).
   Workload screen of mcf (7 Oct, N=1): V0 1.022, V1 1.015 (unresolved), V2 1.034, V3 1.011, V4 1.019.
4. **memcached (4).** Leg mct4 (A/A 0.999): the one configuration with the hand-written EVCONF line, 1.806 over stock
   built by the same root and runtime. A route without annotations whose ownership is checked at run time
   (EVCONF-CHECKED) is in progress; the static spec-free derivation (EVCONF-DERIVE) is parked: it derives 0 fields.
5. **FFmpeg** is carried by the single-threaded stream copy: copy 2.71×, mjpeg 1.04×, h264 1.00×, h265 0.99×.
6. **Stock TSan over native:** SQLite 4.6× and ~19×, FFmpeg 2.8×, Redis 6.0×, MySQL 7.5×, memcached 6.7×
   (8.1× in the 7 Oct workload screen).
7. The submitted paper's figures came from other workloads, the Intel host and a compiler with two elisions later
   found unsound.

## The one configuration

| app | -wp | what it costs against the app's line of record |
|---|---|---|
| Redis | yes (1 derived start routine, 13 lines) | nothing measurable: rcq over the best before 1.036, inside a wide A/A (ccrd4) |
| memcached | yes (EVCONF line; 4 derived start routines, 17 lines) | −1.6 %: mcy over zmr 0.984 (mcxy4; the phase runtime ~1.4 %, compiled phase code ~0) |
| SQLite | no | nothing by construction: threadtest3's .text (3,277,712 bytes) and its 76,093 runtime calls are byte-identical; 1 start-up hook call |
| MySQL | no | nothing by construction (27 start-up hook calls); runtime repeat myj over myr 1.002 (myj4) |
| FFmpeg | no | nothing by construction (≤ 284 start-up hook calls); ffn over ffy 1.004 (ffn4, N=1) |

- **Rule (7 Oct): -wp only where the record uses it**, that is a libevent loop covered by the configuration's EVCONF
  line, or a quiet-thread candidate (a derived `start-routine` line). Decided once per program version from one
  summary step; any other program builds without -wp and without the phase flags. MySQL has a libevent loop (the X
  plugin's `ngs::Socket_events`) but no EVCONF line, so it gets none.
- **Start-up hooks** (R2 option (b), b2c356018845): with `phase_guard=1` the runtime's sync hooks run until the first
  thread creation and then go off; counted with a measurement-only runtime (7017a5f8266f), one run each.
  Evidence: `$EXTRA/wt-dev2-r/hookcnt/`.
- memcached's one configuration has no inline hit test: with it (mcq) it lost ~8 % (cc3mc4c, pgs4).

## Table 2. Optimizations that gain

The first eight rows are single levers over P1-v3; the rows from EA-CONTENTS down are measured on top of their app's
configuration, except LO-OBJ-G (over stock).

| optimization | kind | what it is | SQLite | memcached | Redis | MySQL | FFmpeg |
|---|---|---|---|---|---|---|---|
| **DynSTC-RT** | R | single-thread mode in the runtime: while one thread is alive nothing is recorded, range checks included | ⚪ +0.4 | ⚪ −0.3 | `f` 🔴 −2.2 | ⚪ +0.1 | `f` 🟢 **+12.0** |
| **N1** | S | TSan's "already recorded?" test is inlined; the runtime is called only on a miss | ⚪ −0.8 | ⚪ +0.3 (−5 on the record line) | `f` 🟡 +2.5 | 🟡 −1.7 | `f` 🟢 **+5.0** |
| **N1-ST** | R | N1's inline test is skipped while the thread is in single-thread mode | ⚪ +0.2 | ⚪ −0.2 | `f` 🟡 −1.4 | 🟡 −1.8 | `f` 🟢 **+16.9** |
| **FE-INL** | S | TSan's shadow call stack push and pop inlined | ⚪ −0.7 | ⚪ +0.8 | `f` 🟢 **+4.8** | 🟢 **+3.6** | `f` ⚪ 0.0 |
| **FE-INL + VWIDE-loops** (DE-VERIFIED-WIDE) | R | plus check removal inside loops, verified at run time | ⚪ +0.1 | ⚪ +0.6 | `f` 🟢 **+7.2** | 🟢 **+4.6** | `f` 🟡 +1.8 |
| **FE-SINK** | S | the function-entry call moved to the first point that needs the frame | 🟡 +1.1 | 🟡 −1.7 | `f` 🟢 **+4.1** | 🟢 **+2.8** | `f` 🟡 −1.2 |
| **N1-L** | S | N1 only in loops with at most 20 checks | ⚪ +0.3 | ⚪ −0.9 | `f` 🔴 −2.5 | ⚪ −0.1 | `f` 🟢 **+3.0** |
| **MEMINTR** (MEMINTR-SRC, one-side) | S | a memcpy from a constant or private source has only its destination checked | 🟡 +2.5 | ⚪ +0.9 | `f` 🔴 −2.5 | ⚪ +0.3 | `f` ⚪ 0.0 |
| **EA-CONTENTS** | S | a pointer read from a container no longer makes the container shared | — | 🟢 **+1.4** | — | — | — |
| **SWMR-ROOTS** (SWMR-1) | S | no checks on reads of a global whose only write precedes every reader thread | — | 🟢 **+0.9** | — | — | — |
| **EVCONF** | A+R | objects owned by one event-loop thread are unchecked while a run-time guard holds | — | 🟢 **+31.1**ᵘ with the two rows above | — | — | — |
| **EVCONF-RANGES** | A+R | the guard applied to memset/memcpy on an owned object | — | 🟢 **+18.5** | — | — | — |
| **EVCONF-ARGS** | A+R | the confinement carried into the hash function's key argument (a clone of MurmurHash3) | — | 🟢 **+7.3** | — | — | — |
| **EVCONF-INTERCEPT** | A+R | the guard around memchr, strlen and bcmp on the confined read buffer | — | 🟢 **+7.5** | — | — | — |
| **Quiet threads** (QUIET-THREADS, REDIS-MAIN, AUTO-BIO: silent-thread mode, the Redis phase guard) | R | Redis's main thread, which runs ≥ 99 % of the checks, skips its checks (ranges included) while every other thread is quiet since a release it acquired; the background threads' start is proven by whole-program rules | — | — | 🟢 **+31.9** | — | — |
| **LO-OBJ-G**, generated spec (gen6, gen8) | G+R | objects protected by their owner's lock are unchecked while the thread holds that lock | 🟢 **+15.7** over stock | — | — | — | — |
| **LO-OBJ-G**, spec v7 | A+R | the same with the hand-written spec | 🟢 **+25.0** over stock | — | — | — | — |

- N1-ST is measured on top of N1 + DynSTC-RT; FE-SINK adds nothing on top of FE-INL. On FFmpeg the paper's
  compile-time DynSTC gives +10.1 %, DynSTC-RT +12.2 %, both together +31.9 % (DYNSTC-DIRECT); the compile-time one
  loses elsewhere (Redis 0.945, memcached 0.995, MySQL 0.993).
- Quiet threads with only the rules that check a start routine alone (AUTO) read 0.946-0.962 of the best on Redis.

## Table 3. No gain

| idea | kind | what it is | result |
|---|---|---|---|
| Removal-mode DE (DE-REMOVAL), DE-3R, DE-2R | S | covered checks deleted outright; one range check per loop; adjacent fields merged | ⚪ MySQL +0.3 %, SQLite 0, memcached 0, FFmpeg −0.3 %; 🔴 Redis −3.8 % |
| All-paths DE + cycle cut (DE-ALLPATHS, DE-5) | S | a check is covered if every path to it has a cover; covers kept around loops | ⚪ Redis +1.0 %, SQLite −0.6 %, MySQL +0.1 %, memcached −1.2 % (0 on the record line); ≤ 1.5 % of checks |
| EA-SEND, typed indirect calls | S | `transmit()`'s msghdr writes unchecked (A2 in EA) | ⚪ memcached: mse = mcf (1.019 both, mse4) |
| Loop guard (T8, DE-LC) and the T10 package | S | a loop-invariant access checked once per synchronisation-free stretch; the older lever package | ⚪ Redis +2.2 / +3.1 %, memcached 0 / +1.6 %, MySQL −0.1 / −4.2 %, FFmpeg +1.4 % |
| Levers on top of SQLite's LO-OBJ-G | S | N1, N1-L, MEMINTR, FE-INL | ⚪ FE-INL +4.5 %, MEMINTR +3.5 % (inside a wide A/A); 🔴 N1 −18.5 % |
| The old "same location" rule (SAME-LOCATION, DSL) | U | struct fields cover each other (S), array elements (A), a narrower check a wider one (Z) | 🔴 S loses a real memcached race; upper bounds in table 5 |
| LO-OBJ-ARGS; LO-OBJ-RANGES | A+R | LO-OBJ-G carried into SQLite's record comparison; its guard on memcmp and VDBE memcpy | ⚪ unsound ceiling 1.023, inside the A/A; reaches neither site |
| EA interceptor toggle in per-unit builds | S | EA's libc-call toggle without the whole-program definitions list | ⚪ SQLite 0.977, MySQL 1.003 |
| memcached's flag globals (SWMR-FIELD) | S | unlocked reads of `settings`, `expanding`, `hashpower` | unsound ceiling +4.5 %; no sound route (admin-written, a benign race); the field rule dropped 6 Oct after audit A68 (≈ 0.13 % left) |
| Redis I/O threads' client buffers | — | the I/O threads' checks on the buffers they drain | ⚪ ≈ 0: their cycles are a spin |
| Quiet mode beyond Redis | R | the phase guard on the other four apps | ⚪ ≤ 0.4 % of checks fall in quiet intervals |
| MySQL THD owner reads (THD-READS, X3) | S | a connection's own THD fields read by its thread | ⚪ ≤ 2.35 % of checks |
| Run-time owner tag for one-thread objects | U | skip the owner's accesses to memory only one thread touches | unsound: a skipped access leaves no record for a later foreign access |
| N1-ATOMIC | S | inline test for relaxed atomics | ⚪ SQLite, MySQL ≤ 0.65 % of cycles; Redis +1.6 %, unresolved |
| DE across synchronisation-free calls (nosync census), DE-5…DE-8, IPA-DE | S | finer rules for when a call or a cycle breaks a cover; a callee's check covers its caller's | ⚪ ≤ 2.08 % of checks; −0.8…+1.4 % |
| VWIDE, VWIDE-loops alone | R | run-time verified removal where no check dominates | ⚪ −0.9…+1.3 % |
| FE-hot, hot-list N1 (PGO-PLACE), N1-S, N1-LOOPS-∞ | S | FE-INL or N1 only at hot sites | 🔴 none beats the full version (PGO-PLACE now pursued for compile time) |
| FE-PM, N1-PM, N1b | S | out-of-line entries that save fewer registers | 🔴 MySQL −2 %, SQLite −2…−4 % |
| FE-LAZY, FE-INL-CSE, N1-CSE, LIBCALL-INLINE, N2 | S | lazy entry; one thread-state load per function; compact, libc-call and batched checks | ⚪ ±1 %; N2 as built loses races |
| SUBS, SUBS-SEL | RT | a covering record of the same thread counts as a hit | 🔴 −2…−18 %; the selective form is unsound |
| STC-SUM, STC-TS, STC-WL, libfacts | S | closed-world facts for STC and SWMR | ⚪ < 1 % of checks |
| LTO, Attributor, extra LLVM passes (XPASS, XP2) | S | more optimization before instrumentation | ⚪ nothing; the Attributor miscompiles |
| CLONE-ESC, EA-SLOT (EA-SLOT-CONTENTS), ICALL-A2, returns-fresh, field chase | S | finer escape analysis | ⚪ each < 2 % of checks |
| MYSQL-WP, TOPDOWN-FACTS | S | summaries for MySQL; "argument local in every caller" across units | ⚪ ≤ 0.14 % of checks |
| SWMR-G, PUBLISH-ONCE, thread roles, THREAD-IDS | S | finer may-happen-in-parallel facts | ⚪ each < 3 % of checks |
| NOALIAS, CUSTOM-SYNC, MySQL sysvars, INNODB-LATCH, ODR-TRUST | S | language and library facts | ⚪ each < 3.2 % of checks |
| RT-SYNC, SLOT-CHURN (SID-ALIAS), RANGE-OVERWRITE, RANGE-OWN-FASTPATH | RT | runtime-only changes | ⚪ no effect; RANGE-OWN-FASTPATH ≤ 0.3 % by census, not pursued 6 Oct |
| LO-F, LO-W, LO-B1/B2, sound loop ranges, DE-1, DE-2, DE-4, DE-9 | S | further lock-ownership and DE rules | ⚪ ≤ 1.4 % of checks, or unsound |

## Table 4. Open

| item | kind | app | status |
|---|---|---|---|
| **EVCONF-CHECKED**: the analysis proposes event-loop ownership candidates (no application names); run-time owner checks and a shadow ownership marker make each elision sound, a failed check voids the run (exit 67) | R | memcached | approved 7 Oct; design audit A70 and code audit A70b sound with conditions; not timed. Target: the 1.81× line without annotations |
| **SQGEN-AUTO**: the generator derives the lock API instead of naming it | G | SQLite | gen8 derives enter/leave (P2) and the predicates (P3); audit A61f sound with conditions; same code as gen6 |
| **EVCONF-FIELDS**: the owner's reads of its connection's own fields | A+R | memcached | +3.6 % on top of the EVCONF line; audited; needs the wider P-X86-FD |
| QUIET-FE: function entry and exit not recorded while Redis's main thread skips | R | Redis | FE is 10.3 % of the main thread's cycles; not built |
| DE-AV (SAME-PTR, DE-10): DE's covers checked at run time where no dominance holds | R | SQLite | unsound ceiling +8-11 %; a sound flag reaches 3-17 % of executions (≈ +2-4 % expected); parked |
| LO-OBJ-G on the connection's objects under `db->mutex` | A+R | SQLite | unsound ceiling +5.8 %; needs a second lock type; 0 % reachable in this harness (NULL db->mutex); parked |
| Camera-ready legs | — | all | trees on the camera-ready compiler `tsan-cr-8345a0396fa5`, IR-identical to the measured ones; held for the go |

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

- Most of the one-thread mass is real at run time but not provable statically: heap memory reachable from shared
  structures (EA-HEAP: 78.7 / 7.5 / 34.0 / 41.9 % of the checks of SQLite / FFmpeg / memcached / Redis; the part freed
  in the allocating call ≤ 0.6 %).
- **The old "same location" rule** (U; removal mode, over exact DE in the same leg; last row over upstream):

  | variant | Redis (`f`) | SQLite | memcached (`f`) | MySQL | FFmpeg, checks removed |
  |---|---|---|---|---|---|
  | S: fields of one struct | +8.4 % | +11.9 % | +1.7 % | +4.1 % | 13.7 % |
  | A: elements of one array | +5.9 % | +12.8 % | +3.9 % | +1.1 % | 22.1 % |
  | S + A | +11.5 % | +27.3 % | +6.6 % | +6.0 % | 34.4 % |
  | S + A + Z (narrow covers wide) | +10.6 % | +31.2 % | +6.8 % | +7.9 % | 35.4 % |
  | S + A + Z over upstream | +11.3 % | +35.4 % | +7.3 % | +10.3 % | — |

  S loses a race stock reports in every run: memcached's `do_item_link` writes `curr_bytes` then `curr_items` under the
  stats lock, `clock_handler` reads `curr_items` without it, and under S the first write covers the second. A and
  exact DE lose none. The shadow-proxy rule (RedCard) would remove only 0.1-1.1 % of checks.

## Table 6. Every other optimization tried

| optimization (aliases) | kind | what it is | result | standing |
|---|---|---|---|---|
| **DE** | | | | |
| DE package (T4, T4+E) with DE-B selective peeling | S | merge + loop ranges + selective peeling | with exact DE: memcached +3.6 % (`f`), MySQL +3…+6 %, Redis −2.6 %, SQLite and FFmpeg ±1 % | superseded |
| N3, N4, N8 | S | DE path flags; covers across iterations; trimmed entry/exit | ≤ 1 % each | closed |
| DE-8b | U | C allocation functions treated as synchronisation-free | 0 %, and unsound | dropped |
| relaxed-atomic rule (`-tsan-de-relaxed-atomic-nosync`) | S | relaxed atomics do not break a cover (A8) | SQLite's loop-guard skips 0.74 → ≈ 9.2 % of checks | on in every build |
| pure-asm rule; anticipation, acquire tolerance, available-checks dataflow | S | further cover placements | ≈ 0; ≤ 0.56 % | closed |
| POSTDOM-TERM | S | post-dominance with proven termination | ≤ 0.27 % | in the tree |
| **EA** | | | | |
| U2 | U | named arguments never escape | 0.963-0.996 over T1 on four apps | closed |
| integer-copy closure off (A7) | U | EA without following pointers copied as integers | memcached +0.9, Redis +1.0, MySQL −2.7 % | closure kept on |
| EA-7; the record-compare chain | U | arguments of external or address-taken functions | +5.5 k static sites; bound ≈ 5.6 % of SQLite's checks | shelved |
| EA-WP | S | EA with whole-program summaries | 0 % of the one-thread heap mass | closed |
| EA call-site flags (MAAP-style), OWN-STACK-ENTRY (X2) | S | per call site, pointer-parameter checks removable | MySQL 0.01 % with nocapture, 3.01 % bound; own-stack entry ceiling 24.4 %, unsound without an escape argument | parked 5 Oct |
| **STC, SWMR, LO** | | | | |
| STC-1…STC-4, join-aware STC, STC-CALLEE, STC-STRONG (i) (O-STC), DYN-1 | S | finer single-threaded-context rules | ≤ 2.44 %, mostly ≈ 0 | closed |
| MAIN-ONLY (STC-STRONG (ii)) | S | prove statically that Redis's reads happen only on the main thread | statically impossible; 22 % dynamic ceiling, taken at run time by the quiet threads | closed |
| DYN-2 | R | where DynSTC's guard skips | only FFmpeg's stream copy | closed |
| LO-unwind; LO-C | S | an invoke's unwind edge inherits the callee's acquisitions; POSIX calls transparent to LO | refuted; no gain | closed |
| SQLITE-CW | S | closed world for SQLite | 0.02 % | closed |
| SWMR-H | S | a location written only before its publication | ceiling ≤ +0.2 %; 14 lost-race paths in audit | parked |
| SWMR-ROOTS on SQLite, MySQL, Redis | S | memcached's rule on the other apps | ≤ 2.9 % of checks, ≈ 0-1.3 % reachable | closed 5 Oct |
| G-EA | A+R | escape analysis under LO-OBJ-G's guard | loses 3.7 % | closed |
| LO-OBJ-G spec completion; SQLITE-KEY | A+R | BtShared fields v7 does not name; comparator key and payload | ≈ 1.2 % / ≈ 3 % of checks | not built; parked |
| LO-OBJ-G Pager/Wal roots | A+R | Pager and Wal under their b-tree's mutex | 12.7 % of the locked mass | declined (P-PAGER) |
| SQLite sharable guard; PERELEM | R; S | a per-object guard for the shared cache; per-element locksets | cannot reach the mass; memcached ≈ 2.3 %, provable ≈ 0 | closed; parked |
| TLS-rooted escape analysis (LO-TID) | S | objects reachable only from a `thread_local` root | MySQL ≈ 0.8 % on Release | parked |
| **N1, FE and other inline paths** | | | | |
| N1-ST =miss; N1-ST (b), (c) =fs | R | the flag read only on a miss; hoisted; in the fast state | FFmpeg −6.6 %; equal to the front form | front kept |
| N1-ST-WORKER; N1-SPLIT, N1-PAIR; N1-CSE across calls | S | variants of the inline test | 0; subsumed by DE-2R; ≤ 1 % expected | closed; not built |
| FE-INL exit-max=1; unified exit | S | FE-INL variants | −5.2 % on MySQL; identical | dropped |
| FE oracle; TSan-aware inlining | U | entry/exit removed | Redis +23.5 pp, MySQL +13.6, SQLite +4.4 | ceiling |
| call-cost ceiling (N1/N2-real) | U | run time in the runtime calls of hitting checks | SQLite 21 %, FFmpeg 17.5 %, Redis 25 %, memcached ≈ 0 | bounded N1, N2 |
| FE-NOFRAME | S | no entry/exit in functions without accesses | — | not taken: changes report stacks |
| MEMINTR-INLINE | S | small memcpy/memset inlined | ≤ 1 % | not recommended |
| **Ownership variants** | | | | |
| EVCONF-INTERCEPT write side (X5) | A+R | the guard on sendmsg/writev iovecs | unsound ceiling +0.7 % | closed |
| fresh item until linked; one-sided `_nosrc` copies | A+R | memcached's new item; covered-source copies | ≤ 1.1 %; ≤ 0.8 % | not built |
| quiet-thread refinements | R | the skip bit folded into the fast-state load; re-read only after synchronising calls | ≤ 0.3 % beyond the fold | parked |
| R1-R7 | — | the EVCONF full-connection claim, quiet-mode allocation stacks, skipping the stats mutex, a SQLite connection latch, refcount readers, DOM-FS, guard hoist | unsound, report-changing, or ≤ 0.3 % | rejected 4 Oct |
| OWN-HANDOFF | A | checks skipped while the program's hand-off protocol says one thread holds a buffer | Redis +13.9 %; trusts the program's protocol | replaced by the quiet threads |
| OWN-CONN | A+R | an owner guard on SQLite's private connections | ≈ +21 % estimated | withdrawn (P-CONN not asserted) |
| OWN-STACK; SPIN-ACQ | R; RT | checks skipped on the thread's own stack; a spin of atomic loads acquires only on a change | sound form ≈ −1 %; Redis ≤ +14 % (ceiling) | parked |
| FFmpeg heap hand-off | A | buffers handed between FFmpeg's threads | rests on a program-protocol premise | closed |
| **Runtime-only track** (parked from the paper) | | | | |
| DD-EXACT (DD-COST; A = DD-CHEAP, B1 = DD-BIG), DD-STATIC | RT | TSan's deadlock detector | A sound, null; B1 memcached +21.7 % but drops lock-order reports; DD-STATIC out of scope (6 Oct) | parked |
| RT-SYNC-LF | RT | lock-free acquire loads | unresolved | parked |
| RT-RANGE, RANGE-HIT-SKIP, RANGE-VEC, RANGE-UNIFORM | RT | faster range checks | ≤ 0.5-2 % | closed or parked |
| RT-ALLOC | RT | faster allocation bookkeeping | Redis ≈ 11 % of cycles (mass) | parked |
| RT-SLOT, SLOT-PREF, empty-epoch elision, early reset | RT | slot preemption and global resets | ≤ 1.7 % | closed |
| ALLOC-COVER, RANGE-COVER | RT | allocation and range records covering later checks | 5.6-7.0 % (proxy) | out of scope |

No result yet: sendmsg batching, the interceptor bypass, EA-P1's out-parameter cost, small EA precision counts.

## History: superseded results

| item | was | now | why |
|---|---|---|---|
| Redis quiet threads, hand-written lines | 1.39-1.40×ᵘ (annotation) | 1.54× derived (AUTO-BIO), table 1 (2) | lines derived from the whole program, 5 Oct |
| Redis, derived lines on the earlier root | 1.45× | 1.54× (rcs4, same-compiler stock) | the one configuration, 6 Oct |
| SQLite spec v5, v6 | +18.8 %, +23.5 %ᵘ | v7 1.25× (zg64b) | v7 carries the latch; audited |
| SQLite generated spec gen5 | 1.14× (zg5st4) | gen6/gen8 1.16× (zg64b) | gen6 adds 64 nocopy lines |
| memcached EVCONF line on the camera-ready root (cwn) | +80.3 %ᵘ | 1.81× (mct4) | the one configuration, 6 Oct |
| memcached without annotations | 1.01× (analyses + EA-CONTENTS + SWMR-ROOTS) | 1.02× (mcf, mcnf4) | FE-INL + MEMINTR, N1 off |
| memcached spec derived from the program alone (EVCONF-DERIVE, zma) | 1.010 over the hand spec (zma4) | parked 7 Oct; EVCONF-CHECKED replaces it | audit A59e found it unsound; the fixed derivation derives 0 fields |
| memcached one configuration with the inline hit test (mcq) | 0.922 over zmr | mcy without it (−1.6 %) | N1 costs ~5 % on memcached |
| FFmpeg one configuration with -wp (ffy, ffr) | ffr over zfb 0.991 | ffn without -wp (ffn over ffy 1.004) | -wp rule, 7 Oct |
| MySQL one configuration with -wp (myr, myy) | 0.994 / 0.987 over zyr | no -wp (myj over myr 1.002) | -wp rule, refrozen root |
| SQLite one configuration with -wp (sqx, sqy) | not resolvable on AMD (A/A 0.93) | no -wp, identical code | -wp rule, 7 Oct |
| Removal-mode DE on FFmpeg | +10.2 % (screening) | −0.3 % (full leg) | the screening did not reproduce |
| Compile time, per-unit | SQLite +20 %, memcached +8 %, Redis +9 %, FFmpeg +16 % | whole build, table below | per-unit compiles left out the -wp summary step |
| Compile time, first rows on 744024407b56 | Redis +324 %, memcached +121 % (wall time) | table below | method fixed 7 Oct (`$EXTRA/de-recovery-gate/degen/cctr/btime.txt`) |

## Compile time

Root `tsan-cc-744024407b56` (7 Oct, every compile-time fix in) for every row, against stock TSan built by that root
from the same tree. Compile = the -wp summary step + the build, from build.sh's ns timer; the static site count is
excluded. Fresh trees, no ccache, CPUs 28-35, medians of 3 (reps within 1 %, FFmpeg within 7 %). Rows:
`$EXTRA/wt-dev2-r/cct/btime.txt`.

| app | line of record (tree) | -j | stock | line | overhead | of which summary step |
|---|---|---|---|---|---|---|
| memcached | no annotations: the analyses + FE-INL, MEMINTR (mcf) | 8 | 6.07 s | 11.01 s | +81 % | 4.67 s |
| memcached | one configuration + EVCONF (mcy, the 1.81× row) | 8 | 6.16 s | 14.83 s | +141 % | 8.53 s |
| Redis | one configuration (rcy) | 8 | 5.06 s | 22.91 s | +353 % | 13.06 s |
| SQLite | LO-OBJ-G gen6 (zg6) | 8 | 32.88 s | 41.63 s | +27 % | (no -wp) |
| FFmpeg | ffn (no -wp) | 8 | 77.30 s | 139.29 s | +80 % | (no -wp) |
| MySQL | Release, FE-INL (rpf) | 6 | 858.30 s | 1005.21 s | +17 % | (no -wp) |

- **memcached** has two stock arms: mcf's harness builds the whole target, the one-configuration tree only the server.
  **MySQL** ran at -j6 (8 jobs exceed the 16 GiB scope), from myrpfj6r1, a copy of rpf's tree.
- **Why FFmpeg grows +80 % without -wp:** its line carries N1's inline hit tests; ffmpeg and its 8 libraries have 3.8×
  stock's .text. On 40 random units the analyses alone add 17 % compile CPU, the inline hit test brings it to ×1.96,
  N1-ST adds 10 %. MySQL's FE-INL grows mysqld's .text ×1.86. Evidence: `$EXTRA/de-recovery-gate/degen/n1g/`.
- **Where the summary step goes** (FFmpeg, each opt pass alone on the linked IR): the phase-summary pass took 234.6 s,
  re-reading the module's asm text once per global; fixed in 510cbba6ec58 (249.8 → 37.5 s, identical records).
- **Reductions in progress (7 Oct):**
  - lock-scope memo in the summary step: opt 4.46 → 1.37 s on memcached, 9.14 → 4.52 s on Redis, identical outputs;
  - configure once for memcached's two builds;
  - profile-guided placement of the inline fast paths (PGO-PLACE) for FFmpeg;
  - a single-compile generator for FFmpeg's -wp arms (make 56.5 → 44.9 s at -j16, identical IR) is in the harness
    but on no line of record.

## Notes

- **Method.** Each arm is built at four code offsets; each leg has an A/A arm. Servers and clients run on disjoint CPUs.
- **memcached's workload** (V4) was chosen among five measured over upstream (2 Oct configuration): default input
  +2.8 %, pipelining +3.5 %, 32-key gets +6.3 %, long keys +16.8 %, the chosen mix +30.4 %.
- **Camera-ready compiler:** `tsan-cr-8345a0396fa5`, one compiler for all apps (audits A42-A58b; IR suites, quiet-mode
  tests, check-tsan in 12 configurations, go-check).
- **Race preservation.** Every row of tables 1-2 loses no race stock TSan reports under the premises of
  `soundness-fixes.md` §6: IR tests, check-tsan, reproducers with controls, an independent audit, and application runs
  (10 stock vs 10 optimized runs per app, races matched by location pair; no race-report site lost: memcached 4 kept,
  SQLite 3, MySQL 232-236, Redis all seeded races; FFmpeg's stock reports none).
- **Premises awaiting a ruling** that results of record use: P-X86-FD (narrow form) and P-LIBEVENT (EVCONF),
  A5, A8, A9. MALLOC-ATTR was withdrawn for Redis on 6 Oct. Details: `soundness-fixes.md` §6.
