# TSan runtime ideas: inventory (tsan-rt track)

Canonical list of every runtime-side idea, replacing the earlier ideas.md (its rows are all included here; the old
file is in git history). Plan, decisions and experiments: README.md in this folder.

State 26 Oct 2026. Read-only audit: nothing built or run. Every number below is quoted from the file:line named next to
it; "est." marks an estimate or census, not a timing. Paths are relative to `/home/alexey/atc26-handoff/` unless absolute.
Hosts: `f` = focs (Intel Xeon w9-3495X), `a` = apollo (AMD EPYC 9115).

## 0. Sources used

- `runtime/`: README.md, ideas.md, als.md, dd-exact-ab.md, n1-atomic-a081.md, range-hit-skip.md, sid-alias.md, slot-pref.md
  (the .ru.md files were not needed).
- `ideas.md` (rows RT-SYNC :85, DD-COST :86, N1b :87, N2 :77, N1-ST :81, RUNTIME :84, SUBS :34, SUBS-SEL :35, N6 :41,
  DynSTC :134, the 4 Oct sweep :240-341, X1-X5/R1-R7 :342-447, E1-E15 :448-523).
- `experiments.md` (:185-187, :248-277, :336, :917, :945-961, :979-991, :1007, :1228-1229, :1257-1294, :1876-1906,
  :2568-2576, :2689-2720).
- `optimization-results.md` (§1 :38-120, §3 :199-244, §4 :245-280, §6 :445-700), `soundness-fixes.md` (§4f :353-370,
  §6 :396-470), `paper-results.md` (grep only), `upstream-pr-results.md` (:1-60, :130-218).
- `audits/audit-2026-09-25/A3-runtime.md` (the original runtime audit), `audits/audit-a44-runtime-stock-path.md`,
  `audits/conservatism-review.md` (§1, §6 ranked list :460-500, §7), `audits/audit-a84-atomic-load-skip.md` (via als.md),
  `audits/audit-a34-dd-exact.md` (via dd-exact-ab.md).
- `notes/measurements/hotspots.md`, `notes/measurements/hotspot-time.md` (:120-214), `notes/inline/stock-cost-fixes.md`,
  `notes/inline/atomic-hooks-ideas.md`, `notes/ownership/phase-quiet-rt.md` (:1-60, :313-425),
  `notes/ownership/redis-phase-guard.md`, `notes/ownership/owner-rt.md`, `notes/section3-runtime-forms-draft.md`,
  `notes/appendix-runtime-soundness-draft.md`, `archive/hypothesis-review-2026-10-03.md` (rows 97-106),
  `archive/ideas-30.09`, `tsan-improve-handoff-2026-10-08.md` (:78-95, :196), `README.md`.
- Memory notes: runtime-track-separate, tsan-rt-lane, stated-assumptions, no-lost-races-hard-rule,
  tsan-eviction-not-age-based, own-heap-static-verdict, redis-phase-guard, memcached-evconf-checked, parked-swmr-h-lo-f.
- Code: `/home/alexey/dev/llvm-project-focs-lab` (see §5).

## Soundness classes used in the tables

| class | meaning |
|---|---|
| **EXACT** | same shadow/clocks/reports as stock (bitwise, or the audit's "exact against stock") |
| **EXACT-G** | every granule stock reports is still reported, no later; the report's pair or stack may differ (criterion G, adopted only for SUBS, `soundness-fixes.md:416`) |
| **EVICT** | may lose only races of the bounded-state class the group accepted on 1 Oct: P-EV eviction, P-RESET global reset, P5 recycled trace part / slot choice (`soundness-fixes.md:421-423`) |
| **PREMISE(x)** | needs the named premise x |
| **DD-DIFF** | race detection untouched; the deadlock (lock-order) detector's reports differ from stock |
| **UNSOUND** | an audit or a reproducer showed a loss stock does not have |
| **STRONGER** | changes which races stock already loses (fewer stock losses), not per execution |

## 1. Inventory

### 1a. Runtime-only ideas (compiler-rt only; the binary is unchanged)

| # | name (aliases) | mechanism | status (date) | best measured effect | census / estimate | soundness | code | open / next |
|---|---|---|---|---|---|---|---|---|
| R1 | **RT-SYNC** (S2a, "skip no-op acquires", flag `skip_own_acquire`) | skip the vector-clock join of a mutex acquire when a per-SyncVar last-releaser stamp says nothing was released into it since this thread's own release | built + gated 29 Sep; timed null on all 5 apps; then left (runtime folder still says "parked 25 Sep") | null: Redis 0.989 (A/A 0.992), SQLite 0.997, FFmpeg 1.006, memcached 1.005, MySQL 1.002 (`f`/`a`, experiments.md:1278-1282) | memcached 1.431 G mutex joins, 88.5 % exact no-ops, 75 % stamp-predicted, stamp_wrong 0; ≈0.6 % of server user time (ideas.md:85, experiments.md:277); Redis 5.10 G atomic joins 100 % no-op | EXACT (stamp_wrong 0; lock event, trace, mutex set kept) | branch `experiment/rt-sync` 3da6710f0d43, root `tsan-rtsync-3da6710f0d43`; counters `measure/sync-counters` c8f04181f253 | the join is not where the sync cost is; keep as a negative result |
| R2 | **RT-SYNC S1** (conditional clock stores) | `Release` into a SyncVar stores only the clock lines (vectors) that changed | never built (parked 25 Sep) | — | A3 est. MySQL 3-8 % (read-unlock 22.9 % of cycles with 18 readers looks like line contention) (A3-runtime.md:101) | EXACT (bitwise) | none | perf annotate MySQL `MutexReadOrWriteUnlock` first (A3-runtime.md:188) |
| R3 | **RT-SYNC S3** (one SyncVar lookup per lock) | hand the `SyncVar*` from Pre(Read)Lock to Post; avoid the second SlotLock + GetSyncOrCreate + `s->mtx` that only feed the deadlock detector | never built | — | A3 est. MySQL 2-5 % (PreReadLock 8.4 %, GetSync 4.9 %), memcached 1-2 % (A3-runtime.md:103) | EXACT | none | overlaps DD-EXACT A (R8) |
| R4 | **RT-SYNC S2b / S4** (O(1) release-store by the last writer; shared-mode read-unlock) | (S2b) last ReleaseStore by the same thread with unchanged counter writes only its own entry; (S4) per-entry atomic max into `read_clock` under a shared lock | never built | — | A3: S2 1-3 %, S4 "MySQL reader serialisation" (A3-runtime.md:102, :104) | EXACT / EXACT at the writer's acquire | none | memory-ordering audit for S4 |
| R5 | **RT-SYNC-LF** (runtime form of SPIN-ACQ; flag `skip_unchanged_acquire`) | an acquire load skips the SyncVar lock and the join when the object's clock token (unique per write) equals the one this thread saw at its last acquire; token read before and after the value | built + gated 30 Sep (matrix 12/12 × 344); one timing leg; "pending audit", never audited | Redis `f` 1.025 vs A/A 1.031: NOT RESOLVED (experiments.md:1289) | Redis 1.859 G acquire loads with a SyncVar, 99.990 % skipped (ideas.md:85); SPIN-ACQ context: Redis io-threads 1 vs 20 = 1.144 (experiments.md:185-187); the skipped loads are io-thread polling off the request path (n1-atomic-a081.md:56-59) | EXACT (claimed; no audit) | `experiment/rt-sync-lf` 61af8477c754 (ea791e96cba6), root `tsan-rtsynclf-61af8477c754` | superseded by ALS (R6), which also skips the slot lock |
| R6 | **ALS** (atomic load skip; options `atomic_load_skip`, `atomic_load_census`) | per-SyncVar version word (0 writing, 1 empty, else unique); an acquire-order atomic load that sees "empty" or a version in its 64-entry per-thread cache, and still owns its slot, skips the slot lock, the SyncVar reader lock and the join; the access check stays | built, audited (A84 SOUND after 3 deltas), timed 10 Oct; parked 10 Oct (runtime not to be touched) | Chromium 49 blink rows: +2.1 % `f` (crlh1, experiments.md:2570), +2.2 % `a` (crld2, :2569); parser +12.5-13 %; 17 stories +6.3 % `f` (crefs, :2572); no memory cost | micro 52 → 16 ns per with-sync load, 8 threads on one atomic 2,000 → 110 ns (als.md:12-13, runtime/ideas.md:17); Chromium path 21-32 % of browser CPU (als.md:15-17); 86-90 % of with-sync loads eligible, default options (als.md:46-48) | EXACT (A84; not covered by tests: CAS window, reset racing a load, fibers, fork; als.md:36-40) | `experiment/als` c911b1f1ff67 (5 commits on 8345a0396fa5; options `atomic_load_skip`, `_skip_cache`, `_skip_verify`, `_census`, `_mutant`), root `tsan-als-c911b1f1ff67` | memcached, Redis, MySQL (std::atomic loads 6.5 % of MySQL cycles, 74 % seq_cst, hotspots.md:111); x12 + preservation with the flag on |
| R7 | **SyncVar-pointer cache** (ALS's next step) | cache the meta-shadow → SyncVar walk with an incarnation id | idea (10 Oct) | — | est. 2-4 % of Chromium samples (runtime/ideas.md:18); `MetaMap::GetSync` 6-9 % of Chromium CPU (als.md:16), 2.5 % memcached, 4.9 % MySQL (A3-runtime.md:52-55) | EXACT if the incarnation id is checked | none | design + audit |
| R8 | **DD-EXACT A** (DD-CHEAP; option `deadlock_detector_exact_prelock`) | return early from MutexPreLock/PreReadLock when the thread holds no lock, the SyncVar already exists and the call has no creation flags | built, audits A34/A34b SOUND, gated DDAB5 1 Oct; parked 1 Oct ("deadlock detector, not race cost") | memcached `a` 0.998, `f` 0.988 (null); MySQL `a` 1.041 over its own base (experiments.md:950) but 0.996 over stock on the newer root (:1263) | pre-lock hooks skipped 8,176 per small memcached run (:1257) | EXACT for races and for lock-order reports (dd-exact-ab.md:14-35) | `experiment/dd-exact-ab` 944fb9f4c502, root `tsan-ddab-944fb9f4c502`; first form `experiment/dd-exact` a2a7387e90aa; unbuilt `experiment/dd-cheap` 40788cc8bbf7 | MySQL gain did not recur; low priority alone |
| R9 | **DD-EXACT B1** (DD-BIG; option `deadlock_detector_large_table`) | deadlock-detector node table 16,384 instead of 4,096, so the graph flushes (and the global spin lock) far less | built, gated, timed; parked 1 Oct; lock-order loss "pending decision" | **memcached +19.4 % `a` (1.194, experiments.md:945), +19.8 % `f` (1.198, :1265), +21.7 % `f` record leg (1.217, :1294)**; MySQL 1.038 (:950) / 0.995 over stock (:1263) | DD-COST: detect_deadlocks=0 gives memcached ×1.29, MySQL read_only ×1.38 (ideas.md:86); detector ≈16-18 % of memcached server user cycles (MutexPreLock 7.5-8.0, StaticSpinMutex::LockSlow 5.8-6.9, DD::MutexAfterLock 2.1-2.3) (ideas.md:86); +31 MB RSS (experiments.md:1257) | DD-DIFF: races untouched; adds earlier lock-order reports (Test4) and can lose some stock makes after a flush (A34 H2: recursive rdlock FIXME path, timedrdlock without pre-lock, racing check+insert) (dd-exact-ab.md:37-53) | same root as R8 | the biggest measured runtime-only gain; needs a ruling on lock-order equivalence, or re-checks on the unchecked paths (adds reports) |
| R10 | **DD-STATIC** | skip detector bookkeeping on lock classes no static cycle can reach | not pursued 6 Oct (runtime/ideas.md:62-63) | — | lock-class census written, never built (`experiment/dd-census` 208ace36b1e0, 5e944dac269e; ideas.md:86) | DD exact if the class graph is sound | branches above | compiler + runtime |
| R11 | **RANGE-HIT-SKIP** (RT-RANGE R2) | write the range's trace event lazily at the first missing cell, as stock's MemoryAccess16/Unaligned already do | closed 28 Sep by census | — | all-hit share of range calls FFmpeg 81 %, SQLite 73 %, Redis 50 %, memcached 11 %; bound ≤ 0.53 % (≤ 1.1 % conservative, SQLite) (range-hit-skip.md:144; experiments.md:266) | EVICT (P5: the report's previous access may restore to an earlier event of the same epoch) (range-hit-skip.md:48-73) | census `measure/range-hit-astats` 40be4af6535c | — |
| R12 | **RANGE-VEC** (RT-RANGE R1) | SIMD hit walk, 2 (AVX2) or 4 (AVX-512) granules per compare | not opened | — | hit cell 0.473 ns vs 1.269 as single accesses (range-hit-skip.md:134-140); bound ≤ 0.9 % SQLite, < 0.4 % elsewhere (runtime/README.md:23); A3 est. 2-4 % (A3-runtime.md:105) | EXACT | none | only if ranges are re-attributed (gap G4) |
| R13 | **RANGE-UNIFORM** | on a miss, a middle cell whose four slots equal the previous cell's reuses its CheckRaces outcome and store index | closed 29 Sep | micro: cold fresh shadow −23 % per cell (8.99 → 6.91 TSC cycles), foreign +2 %, below the 30 % bar (experiments.md:270) | memcached 34.6 of 45.1 G missed cells uniform (76.7 %), FFmpeg 93.5 %, SQLite 63.4 %, Redis 2.1 % (range-hit-skip.md:192-197); bound ≈ 2 % of memcached server user cycles (experiments.md:270) | EXACT, no premise, if every cell is still loaded (range-hit-skip.md:151-175) | census `measure/range-uniform-astats` e03e517726c2; micro `/extra/alexey/wt-dev2-r/rangemb` | — |
| R14 | **RANGE-OWN** (RANGE-OWN-FASTPATH) | bulk-store a range whose slots are all empty or own-sid | not pursued 6 Oct | micro: own-sid miss 2.9 ns/cell vs hit 0.53 (range-hit-skip.md:233-237) | bound 0.3 % memcached, 0.2 % Redis (range-hit-skip.md:249-256) | EXACT (stock's slot rule kept) | scratch counters only | — |
| R15 | **Range (d): exact write-range overwrite** | a write cell whose slots all happen before it becomes [W,0,0,0] (SIMD; sid clock cached per call) | not built 1 Oct (below gate) | micro 20.3-22.5 % per cell, gate 30 % (experiments.md:1902-1905) | est. 2-4 % of memcached user (experiments.md:1887) | EXACT-G (granule-level; A3 rejected zeroing slots under the literal criterion, A3-runtime.md:176) | micro only | needs the criterion-G ruling extended |
| R16 | **msghdr batching** (b') | one range per sendmsg msghdr instead of per iovec | parked 30 Sep | — | 0.5-1 % memcached (experiments.md:1888) | EXACT if the same cells are checked | none | — |
| R17 | **RT-ALLOC** | faster freed/reset shadow fill; cache the allocation stack id by call stack (StackDepot::Put) | parked 25 Sep; never profiled | — | Redis ≈ 11 % of cycles (malloc/free shadow 8.2 + StackDepot::Put 3.1, A3-runtime.md:49); Redis main thread malloc/free/realloc 8.2 % (+0.9) on the record config (hotspots.md:75); FFmpeg copy "ranges and allocator 27 %" (hotspots.md:17) | EXACT if the stack id is reused only when the shadow stack equals the cached one (conservatism-review.md:474); deferring Put is UNSOUND for report content (A3-runtime.md:180) | none | perf split of the allocator path, then build |
| R18 | **SLOT-PREF** (RT-SLOT, A3's C2) | FindSlotAndLock prefers an unowned, non-exhausted slot over preempting a live owner | closed 28 Sep, measured | — | only 1.5 % of memcached's sampled preemptions found a free slot (slot-pref.md:185-188) | STRONGER (fewer false-HB preemptions), within P5 | counters `measure/slot-stats` eece0eea7c77, `measure/slot-cost` (root c15e136e6771) | MySQL may differ (threads exit) |
| R19 | **Early reset** (chain breaking) | DoReset when the first preemption chain would start | estimate only (28 Sep) | — | memcached chains ≈ 66 s thread time per run (re-lock 53.4 s, 4.4 µs/step, half waiting for `slot_mtx`) ≈ 1.7 % of server user time (corrected, runtime/README.md:55); resets ×1.26 (+120, +1-2 s); MySQL chains ≈ 125 s, resets 129 ms each (slot-pref.md:169-215) | EVICT/STRONGER: P5/P-RESET cover the timing; net detection effect (more wipes, fewer false HB) needs a preservation run (slot-pref.md:212-214) | none | preservation run before any timing |
| R20 | **Empty-epoch release** (epoch budget) | a release publishes e−1 and does not increment when nothing at all was recorded in epoch e | census 28 Sep; not built | — | memcached 12.2 % of 1.891 G epoch increments close a truly empty epoch (likely PTHREAD_MUTEX_INITIALIZER mutexes) → ≈ 12 % fewer preemptions and resets, ≈ 8 s of 66 (slot-pref.md:190-199); MySQL 0.05 %, FFmpeg/SQLite ≈ 0 | EXACT when the flag is set at every record (hit or miss) (slot-pref.md:87-93); retagging sync accesses UNSOUND (slot-pref.md:100-106) | census on `measure/slot-cost` | small speed effect; detection-side benefit |
| R21 | **Lazy epoch increment** (SID-ALIAS variant C; conservatism-review #12) | defer the post-release EpochInc to the thread's next access check or epoch-carrying event (pending fast-state bit fails every hit test) | census 1 Oct; not built | — | releases with no check since the last bump: memcached 229 M of 1.89 G (12.1 %), MySQL ≈ 0, Redis 40 k (sid-alias.md:93-97) | EXACT (deferred form; the naive skip is UNSOUND, sid-alias.md:75-79) | census `measure/range-census` 8b5653e84bb0 | same mass as R20; pick one |
| R22 | **SID-ALIAS A / H** | (A) treat records of the thread's own older sids as own in CheckRaces (rewrite instead of evict); (H) accept them as hits when E is the last epoch and no release since | A and H proposed closed 1 Oct | — | memcached 55.5 G of 91.5 G checks (60.6 %) miss against an own-old sid; H eligible 0.003 %; A bound ≈ 0.7 % of server cycles (sid-alias.md:93-103; experiments.md:1905-1907) | A: EVICT (eviction indices change; audit owed); H: EXACT under F1-F3 + no-release | none | closed; the churn needs a shadow-format change (R24) |
| R23 | **Eviction victim policy** (A3 E1, E2; ideas N6) | E1: evict a slot subsumed by `cur` or another slot first; E2: evict own non-current-epoch records first | never censused or built | — | memcached discards concurrent foreign plain history at 7.9 % of all checks; evictions 34 % of checks (A3-runtime.md:88-95) | E1 EVICT and never worse than stock's pick; E2 EVICT | none | detection lever (fewer eviction losses), plus E2 speed if thrash misses exist |
| R24 | **Wider epochs / more sids** (shadow format change) | more epoch bits or slots per reset cycle, so resets and preemption chains stop | rejected in A3 as out of reach (A3-runtime.md:178); named as the only real fix of the churn (sid-alias.md:101-103) | — | memcached burns ≈ 3.9 M epochs/s against 256 × 16 k per cycle; 456 resets, 14 M preemptions per run (sid-alias.md:4-7; slot-pref.md:9-17) | STRONGER (fewer stock losses) if exact otherwise | none | the runtime paper is the place for it |
| R25 | **RT-STOCK-COST** (runtime parity; two-archive runtime) | keep every research hook off the stock path: TLS placeholder hidden (no GOT loads), flag tests moved to miss paths, separate `checked` archive for marker code | partly done (A44 fixes, TLS fix 302d070bf406 in 8345a); two-archive plan postponed 8 Oct | Redis TLS effect +3.1 % cycles at equal instructions, fixed (phase-quiet-rt.md:408-413); research counters had made stock 1.4× slower (soundness-fixes.md:362); code alignment FFmpeg +1.3 %, memcached −1 % (hypothesis-review row 105) | residual camera-ready gap: upstream stock faster than the root's stock by 4.7-4.9 % on memcached (optimization-results.md:80, :93), 2-4 % Redis (:74, :76), 1-2 % FFmpeg (:83, :85, :88), SQLite unresolved (:77), MySQL 0.1 % (:72); marker code cost stock memcached 4 % (Delta 1: 0.960, tsan-improve-handoff-2026-10-08.md:83-88) | EXACT (behaviour-preserving) | stock-cost fixes in 8345a: 945c0c917392 (one mask test), 234da012bff3 + a730bf95356e (eviction bump and `__tsan_shadow_generation` removed), 428767f7fb34 (EA toggle via `ignore_interceptors`), 302d070bf406 (TLS hidden), 2c6e60e2e64d (LO-OBJ-G hooks gated), 559eacf4671f (entries back to upstream form). Still on the stock path at 8345a, by reading: one extra mask test before the hit test in MemoryAccess/16/Unaligned/RangeT (`tsan_rtl_access.cpp:599`), a `phase_skip` load in MemoryAccessRangeT (:1018), `phase_quiet` loads on free/imitate-write (:882, :936, :949), `phase_on` tests on every mutex path, a `LoObjEnabled` load per lock/unlock/free, a seq_cst RMW on `__tsan_active_thread_count` per create/join; two-archive branches `experiment/evconf-e52d-merge` (not frozen) | a runtime paper must start from a runtime equal to upstream; settle A44's "upstream instruction count" with a disassembly diff of `__tsan_read4` |
| R26 | **DynSTC-RT** (option `dynstc_rt`) | while one thread is alive the runtime records nothing; the compiled calls still happen | in the ATC'26 camera-ready (adopted for FFmpeg) | FFmpeg `f` +12.0 % (optimization-results.md:208); the option on the stock binary 1.138 (:90); Redis −2.2 %, others ±0.4 % | DYN-2: the gain is FFmpeg copy (single-threaded run) (ideas.md:103) | PREMISE(A10, A11) + the join-without-acquire fix (soundness-fixes.md:357, :409) | 8345a: flag `tsan_flags.inc:98`, `ThreadState::dynstc_single` `tsan_rtl.h:229`, set/clear `tsan_rtl_thread.cpp:132,185,391-401`, test `tsan_rtl_access.cpp:41`; commits c9a1576b3d2c, 57e20e81aa62, 559eacf4671f, e414179bfdeb, 2ae4e920f4ca | already published; not new for a second paper |

### 1b. Runtime parts of compiler levers (compiler + runtime)

| # | name (aliases) | mechanism | status (date) | best measured effect | census / estimate | soundness | code | open / next |
|---|---|---|---|---|---|---|---|---|
| C1 | **N1 miss entries** (`__tsan_*_miss2`) | inline hit test in the program, runtime only on a miss | in the ATC paper | FFmpeg `f` +5.0 %, Redis +2.5 %, MySQL −1.7 % (optimization-results.md:210) | 133-140 B of code per check | EXACT | 8345a: `tsan_rtl_access.cpp:1158-1174`, `MemoryAccessGivenTest` :616, `CheckInlineHitTestAbi` :642 (run at every start, `tsan_rtl.cpp:831`); a449c91338f3; flag `-tsan-inline-hit-test` | — |
| C2 | **N1b** (`__tsan_*_pm`, preserve_most) | keep the call, hand-written hit path, few saved registers | closed 25 Sep: no gain | SQLite −3 pp, memcached −1 (ideas.md:87) | a `_pm` miss +22 instructions | EXACT | 8345a: `tsan_rtl_amd64.S:233-339`, miss paths `tsan_interface.inc:130-175`; 1a74a303485f | — |
| C3 | **N1-ST, N1-ST (c)** | skip the inline test when DynSTC-RT's single flag is set; (c) mirror the flag in fast-state bit 30 | in the ATC paper (FFmpeg) | FFmpeg `f` +16.9 % over N1 + DynSTC-RT (optimization-results.md:211) | — | EXACT under DynSTC-RT's premises (A15, A20b) | 8345a: `FastState::single_` `tsan_shadow.h:75`, `SkipsAccesses` :58, layout CHECK `tsan_rtl.cpp:818`; 98f2d1d301f7, a97cc0d17437, 945c0c917392, 0d09b7502b28 | A44 M1: bit positions wrong on big-endian targets |
| C4 | **N1-ATOMIC** (`__tsan_relaxed_atomics_init`, `pending_signals` at byte 48) | inline hit test on relaxed atomic loads/stores | closed by timing; not in 8345a | Redis +1.1 % `f` / +1.6 % `a`, not resolved; SQLite 0.991-0.996 (experiments.md:917, :957, :1229) | atomic hits 88 % of Redis runtime entries (experiments.md:1891-1896) | EXACT with the pending-signal check (n1-atomic-a081.md:8-11); PREMISE(A8) | `experiment/n1-atomic-hot` e6b97f0ebc66 (gated tip 6c762705beed; 422b08b047ca, f28aa8b2fb85, 95b13077287d) | MySQL Release relaxed share unmeasured (ideas.md:257) |
| C5 | **N2** (`__tsan_check_batch*`) | one runtime call for a sync-free run of checks | not a candidate 25 Sep | best variant −2.7…+1.3 % | — | UNSOUND as built (stores members' records early); the exact form saves nothing (ideas.md:77) | still in 8345a (flag-off): `tsan_rtl_amd64.S:361-561` (33 `__tsan_check_batch_<sig>`), `CheckBatchEntryAbi` `tsan_rtl_access.cpp:1122`; 1c072f13bf20, 2ef2c547d343 | dead code in the camera-ready runtime |
| C6 | **LIBCALL-INLINE** (`__tsan_{memcmp,bcmp,strcmp,...}_light`) | cheaper memcmp/strcmp entries | closed 28 Sep; not in 8345a | saves 3-6 ns per call, ≤ 0.8 % (experiments.md:255) | — | EXACT | `experiment/libcall-light` 6a1b9a0faf05 | — |
| C7 | **FE-INL** (runtime ABI only) | inline shadow-stack push/pop and trace event | in the ATC paper | Redis `f` +4.8 %, MySQL +3.6 %, Chromium CR1 +9.8 % `f` (optimization-results.md:213, :238) | FE 10.3 % of Redis main, 12.1 % of MySQL (hotspots.md:42-46) | EXACT | 8345a: only `CheckInlineFuncEntryExitAbi` `tsan_rtl_access.cpp:699`; d1904c01d71f | — |
| C8 | **FE-SINK** (`*_miss2_fe`) | sink the entry to the first point that needs the frame | in the tables | Redis +4.1 %, nothing on top of FE-INL (optimization-results.md:215) | — | EVICT (P5) | 8345a: `tsan_rtl_access.cpp:1184-1209`; f0e4884e1132 | — |
| C9 | **FE-LAZY** (`__tsan_func_entry_lazy`) | the runtime emits the reconciling FuncEntry/Exit just before any traced event | closed by timing 30 Sep-1 Oct | MySQL `f` L/B 1.009, Redis 1.002 (experiments.md:949, :961) | — | EVICT (P5) | `experiment/fe-lazy` b6c5417665fd (50456473f997) | — |
| C10 | **FE-PM** (`__tsan_func_exit_pm`) | out-of-line preserve_most entry/exit | closed 2 Oct | MySQL `f` −11 % (pp over pb 0.887, experiments.md:988), `a` 0.981 (:991) | — | EXACT | `experiment/fe-pm` c998c9c9a1a7 | — |
| C11 | **MEMINTR** (`__tsan_memcpy_nosrc`, `_nosrc`) | one-sided range check of a copy with a private source | in the ATC paper | SQLite +2.5 %, memcached +0.9 % (optimization-results.md:216) | — | EXACT under EA (conditions K1-K3) | 8345a: `tsan_interceptors_memintrinsics.cpp:53-74`; 7050968e07af, 83e1a20d20cf | — |
| C12 | **KEEP-ON-RACE** (`__tsan_write*_keep`, `kAccessKeepOnRace`) | after reporting a race on a covering write, keep that write's record (stock clears the granule's history) | in 8345a since 2 Oct, emitted by removal-mode DE; being ported into upstream PR llvm#169897 (26 Oct) | DE + fix keeps DE's machine code; timing from 26 Oct (upstream-pr-results.md:19, :201) | — | restores detection DE loses with stock's clear-on-report (upstream-pr-results.md:130-163) | 8345a: `tsan_defs.h:180`, `StoreAfterRace` `tsan_rtl_access.cpp:290` (used :356, :534), entries `tsan_interface.inc:256-300`; 05efa6607c06, 38639200275c (2 Oct) | relation to P-REPORT (K13) |
| C13 | **SUBS / SUBS-SEL** | subsumption-aware hit + absorbing store (global or per merged group) | closed 27/30 Sep | a loss on every app: SQLite 0.818, FFmpeg 0.853; the runtime feature alone costs mjpeg ~22 % (ideas.md:34) | — | SUBS: EXACT-G with open gaps F2/F4; SUBS-SEL UNSOUND (A30 F1, ideas.md:35) | `experiment/subsumption-hit` 7c0a7c968e2e (flags `subsumption_hits`); `experiment/subs-selective` add8aa7bfe27 (never built) | — |
| C14 | **ALLOC-COVER, RANGE-COVER** | the malloc imitate-write (or memset/memcpy destination) record covers the allocating thread's later same-fast-state narrow accesses | never censused; out of scope 4 Oct ("stock would gain equally") | — | proxy: heap written then published 5.6 % of Redis's, 7.0 % of FFmpeg's checks; est. +1…+3 % (conservatism-review.md:302-330, :471) | EXACT-G (+ no-UAF) | none | a runtime paper can take it: stock gains too |
| C15 | **QUIET-THREADS** runtime core (phase guard: quiet periods, acknowledgement, traps, exit-66/67 violations; `phase_guard`, `phase_guard_ranges`; R3 skip entries; PHASE-HIT-FIRST) | the main thread skips checks while every other thread is quiet since a release it acquired | in the ATC paper (Redis) | Redis +31.9 % (optimization-results.md:224); DynSTC column with quiet threads 1.139 (:75) | reach outside Redis ≤ 0.4 % of checks (hotspot-time.md:131-135); `phase_guard=1` costs FFmpeg h264 5-6 % (optimization-results.md:87); guard code ≈ 6 % instructions on Redis (phase-quiet-rt.md:313-317) | PREMISE(A3, A12-DATA, FIELD-ADDR, P-DICT, ...) (soundness-fixes.md:407-415) | 8345a: new file `tsan_phase.cpp` (+1213; entries :1014-1206, stats :919), flags `tsan_flags.inc:103-121`, ThreadState fields `tsan_rtl.h:311-324`, hooks in mutex, atomic, interceptor, thread and DoReset paths; ~44 commits f19bc7dcfb48..2edcfa3b31d6 + c12141e6707b, 281382f2c70c. Not in 8345a: R2 (phase_done, f9c6e72bcc5e), R3 skip entries `__tsan_phase_{read,write}N` (023782c6031b), PHASE-HIT-FIRST (05e767cd51ee), self-off (`experiment/phase-hooks-off` 7ad8d3c7e668), all on `experiment/cr-phase-asm` ab32a9bea55e | QUIET-FE (Redis main FE 10.3 %, ideas.md:248) open |
| C16 | **EVCONF-CHECKED / OWNER-RT / ownership marker** | owner words and a shadow marker; a checked foreign access to marked memory voids the run (exit 67) | in the ATC paper (memcached), measured on root `tsan-ecc-b54ce0196884`, a descendant of 8345a | memcached 1.80-1.82× over upstream (optimization-results.md:46, :93) | marker code cost stock memcached 4 % in a shared runtime (tsan-improve-handoff-2026-10-08.md:87) | PREMISE(IN-BOUNDS, ALLOC-NOUAF) + void runs | NOT in 8345a. `experiment/owner-rt` 2a14a2fbf2d5 (`__tsan_owner_take/check/clear`), `experiment/evconf-checked-rt` 5fb6c13bcd60 (`__tsan_owner_attach/check_block/check_cursor/release/take_slots`), `experiment/fd-reuse` 3a1bf6d1422f (`tsan_fd.cpp` descriptor model); record root b54ce0196884 | two-archive runtime (R25) |
| C17 | **LO-MARK** runtime (`__tsan_lo_mark`, `_release`, `_retire`, `_remark`, `_bind`, `_unheld`; `__tsan_carve*`, `__tsan_bound_fail`) | lock marker in the object's first shadow cell, set by a holder, tested inline | in the ATC paper (SQLite), root `tsan-lomark-913cae99c263`, a descendant of 8345a | SQLite 1.23-1.28× (optimization-results.md:70) | — | PREMISE (16 premises, 5 checked at run time; optimization-results.md:680-689) | NOT in 8345a. `experiment/lo-mark` 718c7520052c (intro a442f255367d, +5012 lines in 22 files), `experiment/lo-mark-v2c` 36c805dd1906; record root 913cae99c263 | — |
| C18 | **DE generation guard** (`__tsan_shadow_generation`) | removal guarded by a shadow generation bumped at resets | rejected 25 Sep | — | foreign current-epoch evictions 6.3 % of SQLite checks (ideas.md:28) | — | added c499ea4253f9, bump dropped 234da012bff3, counter removed a730bf95356e (all inside 8345a's history); old form `fix/de-generation-guard` 0a4c719c7bd7 | none (A44 L1 is resolved at 8345a) |
| C19 | **HIT-PSEUDO** | inline hit test as an X86 pseudo with a register-preserving miss entry | idea 12 Oct (compile time) | — | −21 % (Redis) / −33 % (FFmpeg) of the heaviest units' compile time, as a bound (runtime/ideas.md:65-70) | EXACT | none | compile time, not speed |
| C20 | **OWN-STACK** (published watermark) | skip checks on the thread's own stack below a watermark | parked 3 Oct | unsound ceiling +5.3 % MySQL; sound form ≈ −1 % (ideas.md:197) | — | sound form needs EA | — | X2 OWN-STACK-ENTRY (ideas.md:373-387) |
| C21 | **LO-OBJ-G / G-EA run-time tests** (`__tsan_holds_mutex`, `__tsan_lo_obj_{latch,held,enable,private_enable,held_or_private}`) | run-time "this thread holds the object's lock" and privacy tests behind a spec; per-thread owner caches | in 8345a (SQLite hand/generated spec lines of the paper) | LO-OBJ-G +25.0 % hand spec, +15.7 % generated (optimization-results.md:225-226) | — | PREMISE(R3, P-OWN) | 8345a: `tsan_interface_ann.cpp:469-816`, caches `tsan_rtl.cpp:66-114`, `MutexSet::HoldsWrite` `tsan_mutexset.h:51`, hooks `tsan_rtl_mutex.cpp:75`, `tsan_mman.cpp:271`; b7a932fdd8d6 … 2c6e60e2e64d (11 commits) | costs stock one load + branch per lock, unlock and free |
| C22 | **DE-MERGE entries** (`__tsan_{read,write}{2,4,8}_merged`) | one check for adjacent merged fields | flag-off in 8345a; DE-2R gained nothing outside FFmpeg (ideas.md:206) | — | — | EVICT (P-EV-DEFER for the relaxed form) | 8345a: `tsan_interface.inc:312-344`; 05efa6607c06 | — |
| C23 | **DE loop ranges** (`__tsan_{read,write}_range_all`, `MemoryAccessRangeAllT`) | one range check per sync-free loop | flag-off in 8345a; no gain (ideas.md:174) | — | — | EVICT / P-EV-DEFER, ASYNC-TERM, FWD-PROGRESS | 8345a: `tsan_interface.inc:244-248`, `tsan_rtl_access.cpp:1072`; 66b6fdc302f9 | — |
| C24 | **Active-thread counter** (`__tsan_active_thread_count`) | runtime side of the compile-time DynSTC guard | in 8345a, always on | DynSTC (compiled guard) FFmpeg `f` +10.1 %, Redis −5.5 % (optimization-results.md:209) | — | PREMISE(A10) + shape-36 fix | 8345a: `tsan_rtl.cpp:39`, `tsan_rtl_thread.cpp:128, 358-391`; 6c928316fc25 … d54a22baed6a | seq_cst RMW per create/join on the stock path |

### 1c. Related projects (branch folders meta-shadow/ and l0-filter/, audited 26 Oct; details in projects/)

| # | name | mechanism | status | best measured effect | soundness | relation to our rows | next |
|---|---|---|---|---|---|---|---|
| P1 | **Shadow-resident filter** (meta-shadow) | compiler-inlined copy of TSan's 4-slot hit check, runtime on a miss; leaf functions only; fast state from a runtime call per entry; shadow constants exported (portable) | built, no negative tests | NPB BT ~ −30 % CPU, private loop −69 % (AArch64 VM); Redis/SQLite/memcached/FFmpeg ±1 %; MySQL −2.6 % TPS; no A/A | EXACT | same mechanism as C1 (N1); new: portability (AArch64, Darwin), slot-0-first order | lift the leaf-only rule, `_miss` entries, start-up layout check: a portable N1 for upstream; negative tests |
| P2 | **Parallel note** (meta-shadow) | one exact shadow word per heap cell in a parallel region | built; slower everywhere outside a private loop (Redis −31 %, SQLite +72 % time, BT +253 %) | private loop +27 % | UNSOUND as built (free > 1 KB drops notes; hand-over stores before its CAS) | C14 ALLOC-COVER (its only effective case), C16 | fix the two holes, keep as a negative result |
| P3 | **Workload profiler** (meta-shadow) | hits by slot, private-phase share, shadow-page density | built (diagnostic) | — | n/a (one flag test per access when off) | P1 phase census, R18/R22 | run on our 5 apps for the motivation figure and C14's size |
| P4 | **Fifth shadow slot** (meta-shadow, archived design) | a victim word per cell, consulted only on a miss: fewer eviction losses than stock | design only | — | STRONGER | R23, R24, phase P4 | build in the detection story |
| P5 | **L0 filter** (l0-filter) | per-thread 4096-entry TLS table (address tag, 15-bit local epoch, write bit) checked inline before every access; epoch bumped at atomics and ~26 interceptors | built (on by default; does not compile on Linux; no tests) | call count only (macOS, -O0) | UNSOUND as built: sub-cell bytes, second cell of wide accesses, releases it does not see (sem_post, pipes, annotations), ignore regions, use-after-free, global reset and slot preemption, fibers | DE, DE-LC/DE-AV, C18, C1 (N1), R5/R6 (how ALS keeps a per-thread cache exact) | tag entries with TSan's fast state, key on exact cell/bytes/type, global generation at reset and free; then measure against N1 |

### Notes on the rows

- **R1/R5/R6 are one line of work.** RT-SYNC skipped only the join and kept SlotLocker and the SyncVar lock; it read
  null everywhere. conservatism-review §7.3 points out that this does not bound the lock/slot cost, which is what
  RT-SYNC-LF and ALS skip. ALS is the only one with an audit and a resolved gain, and only on Chromium.
- **R9 is the largest runtime-only number in the folders** (+19-22 % memcached, three legs on two hosts), but it is a
  deadlock-detector change: race reports are untouched, lock-order reports are a superset except the A34 H2 cases.
  The same lever is invisible on the record workload profile of 3 Oct (deadlock detector 2.0 % of memcached cycles,
  hotspot-time.md:183-186), so it must be re-measured on the record workload before it is quoted.
- **Range family (R11-R16).** All closed against per-cell costs from warm or streaming micro-benchmarks, while perf
  puts `MemoryAccessRangeT` at 11.15 % of memcached's server user cycles at ≈ 21 cycles per missed cell
  (experiments.md:269). The difference is cold shadow and TLB misses on fresh buffers (experiments.md:270), which
  none of the closed ideas touches (gap G4).
- **Slots and epochs (R18-R24).** Each cheap exact lever is ≤ 12 % of the churn; speed effects are ≤ 1-2 % of server
  CPU on memcached. Their value for a paper is detection: every preemption is a false happens-before edge (14 M per
  memcached run) and every reset wipes all shadow (456 per run) (slot-pref.md:56-63).
- **R26, C1, C3, C7, C8, C11, C12, C15-C17, C21 and C24** are in the ATC'26 paper or its tables; a runtime paper can
  build on them but cannot claim them as new. C15-C17's runtime parts are on descendants of 8345a, not in 8345a itself.

## 2. Ranked shortlist for a runtime paper

Score = expected dynamic effect across apps × soundness/acceptability × (1 / cost to finish).

| rank | idea | effect (evidence) | soundness | cost to finish | why here |
|---|---|---|---|---|---|
| 1 | **ALS** (R6), with RT-SYNC-LF (R5) folded in | apollo 30 Oct (lazy versions, iNN): Redis +21 % GREEN; memcached, MySQL, FFmpeg, SQLite null; Chromium +2.1/+2.2 % over 49 rows, parser +13 %; 52 → 16 ns per load | EXACT, audited (A84) | low: rebase c911b1f1ff67 into the new repo, x12, preservation, 3 app legs | built, audited, exact, broad |
| 2 | **DD-EXACT B1 + A** (R8, R9) | apollo 30 Oct (iNN, record workload): memcached +3.0 % GREEN; MySQL, Redis, FFmpeg, SQLite null; the earlier memcached +19.4…+21.7 % (3 legs, 2 hosts) did not reproduce | races EXACT; lock-order DD-DIFF (superset except A34 H2) | low (built, gated); a ruling on the lock-order criterion, or a "superset" fix on the unchecked paths | largest measured runtime-only gain |
| 3 | **Mutex sync path S1 + S3 + S2b + SyncVar cache** (R2-R4, R7) | memcached mutex lock/unlock 24.7 % of cycles on the record config (hotspot-time.md:183); MySQL read-unlock 22.9 %, PostReadLock 9.0, PreReadLock 8.4, GetSync 4.9 (A3-runtime.md:54-55); A3 est. MySQL 5-15 %, memcached 2-6 % | EXACT (bitwise) | medium: perf annotate first (RT-SYNC's null shows the join is not the cost) | the biggest untouched cost family |
| 4 | **RT-ALLOC exact stack-id cache + freed-fill** (R17) | Redis ≈ 11 % of cycles (A3), 8.2 % of the main thread on the record config; FFmpeg copy allocator | EXACT (cache keyed by the full shadow stack) | low-medium | cheap, exact, a different app mix |
| 5 | **Epoch budget: empty-epoch release or lazy increment, plus early reset** (R19-R21) | ≤ 1-2 % speed on memcached, MySQL resets 26 s + stalls per run; detection: fewer preemptions and wipes | EXACT (R20/R21); R19 needs a preservation run | medium | the detection half of a runtime paper |
| 6 | **Wider epochs / more sids** (R24) | removes resets and chains (memcached 456 resets, 14 M preemptions per run) | STRONGER; format change | high | the structural fix behind ranks 5 and R22 |
| 7 | **Range: cold-shadow cost** (gap G4) + R15 + R12 | memcached ranges 11.15 % of server user cycles (perf); ranges+interceptors 31.3 % memcached, 36.4 % Redis main, 17.4 % SQLite, 26.8 % FFmpeg copy (hotspots.md:42-50) | R12 EXACT, R15 EXACT-G | medium | big family, but every exact per-cell lever measured ≤ 2 % |
| 8 | **ALLOC-COVER / RANGE-COVER** (C14) | est. Redis, FFmpeg +1…+3 % | EXACT-G | medium (needs one pass hook) | stock-equal gains were out of scope for ATC, in scope now |

**Stories that hang together.**
1. *Synchronization path* (ranks 1-3, plus RT-SYNC's null as the motivating negative result): "TSan spends 25-55 % of
   lock-heavy servers' cycles in sync machinery (A3-runtime.md:28-29); skip the work that provably cannot change the
   detector's state". Apps: memcached, MySQL, Redis, Chromium. Exactness proofs: A84 (ALS), A34/A34b (DD A).
2. *Slots, epochs and resets* (ranks 5-6, R18, R22, R23): both speed and detection; the only story where the runtime
   can report races stock misses (fewer false-HB preemptions, fewer wipes, lossless eviction victims).
3. *Allocation and range shadow traffic* (ranks 4, 7, 8): fresh-memory handling (malloc imitate-write, freed fill,
   cold range shadow).
The single-thread / quiet-phase story (DynSTC-RT, quiet threads, QUIET-FE) is already in the ATC paper and reaches
little outside Redis and FFmpeg copy (hotspot-time.md:131-135).

## 3. Gaps: runtime costs no idea addresses

| # | cost | evidence | what is missing | new? |
|---|---|---|---|---|
| G1 | Mutex lock/unlock path as a whole (slot lock + SyncVar lookup + clock copy + trace + DD hooks per op) | memcached `pthread_mutex_lock` 17.1 + unlock 7.6 % of cycles (hotspot-time.md:183); A3: SlotLock/Unlock 9.1, Mutex Pre/Post/Unlock 10.9, clocks 3.7, GetSync 2.5 (A3-runtime.md:52); release ≈ 400 instructions / 145 cycles (A3-runtime.md:76-78) | only the join (RT-SYNC, null) and the DD pre-lock (R8, null) were tried; an owner re-lock fast path (worker stats mutex ≈ 6.4 %, hotspots.md:68) is listed only as parked "owner-to-owner path (RT)" | partly (S1-S4 named in A3, never built) |
| G2 | Vector-clock representation (256-slot copy/join on every release/acquire) | MySQL read-unlock 22.9 % (A3); Chromium join 1 % (als.md:16) | tree clocks (sublinear join/copy; ASPLOS 2022), dirty-line tracking (S1 is a weak form), adaptive per-SyncVar representation | **new** (tree clocks not in any folder) |
| G3 | DoReset stop-the-world and shadow re-faults | memcached 456 resets × 5.2 ms + 1.99 M minor faults per run; MySQL 204 × 129 ms = 26.3 s + stalls in 55.5 s of contended SlotLock waits (slot-pref.md:169-204) | an incremental or concurrent reset (generation-tagged shadow, lazy wipe per region, madvise in the background); only "early reset" and "fewer increments" exist | **new** |
| G4 | Cold-shadow / TLB cost of ranges on fresh memory | 11.15 % of memcached server user cycles in MemoryAccessRangeT, ≈ 21 cycles per missed cell vs 9 in a cold streaming micro (experiments.md:269-270); "a page-level 'fresh since reset' test would be a new idea" (:270) | page-level fresh-shadow test, huge pages for the shadow mapping, shadow prefetch for ranges | **new** (named once, never pursued) |
| G5 | TSan's internal lock contention and futex waits | memcached server sys +764 s (+46 %), 71.3 M vs 7.4 M voluntary switches (experiments.md:277); half of a chain step waits for the global `slot_mtx` (slot-pref.md:180); DD global spin lock 5.8-6.9 % (ideas.md:86) | a lock-free slot queue, a sharded or lock-free DD graph | **new** (B1 only enlarges the table) |
| G6 | Interceptor entry cost (`ScopedInterceptor`, ignore tests, per-call range checks) | Redis ScopedInterceptor 3.3 %, SQLite 1.1 % (A3-runtime.md:46-49); memcached memchr 2.1 + strlen 1.6 + bcmp 1.0 % (hotspots.md:64) | a lighter interceptor prologue; only compiler-side EVCONF-INTERCEPT exists | **new** |
| G7 | Unaligned and 16-byte access entries (no inline test) | zstd reads mostly through unaligned helpers, 8,131 call sites N1 does not cover (optimization-results.md:277); Redis unaligned 197 M runtime entries (experiments.md:1891-1896) | an unaligned fast path in the runtime (two-cell hit test without the trace) | **new** |
| G8 | Report path when races are frequent (RestoreStack trace replay, `suppress_equal_stacks`) | MySQL ~12,000 reports per run (upstream-pr-results.md:192-193); every timed leg runs `report_bugs=0` (A3-runtime.md:161) | no profile of a reporting run exists | **new** (cost unknown) |
| G9 | Atomic RMW/CAS and release stores | MySQL `compare_exchange<thread::id>` 46 M, `atomic<ulong>::store` 41 M events (atomic-hooks-ideas.md:15); atomics + sync 14.8 % of MySQL (hotspots.md:46) | ALS covers loads only; nothing for release-store or RMW (S2b would) | partly |
| G10 | Function entry/exit trace events in the runtime | Redis main 10.3 %, MySQL 12.1 % (hotspots.md:42-46); FE ceiling Redis +23.5 pp, MySQL +13.6 pp (ideas.md:29) | FE-LAZY failed; a compact FE trace encoding or event-free frames (conservatism-review #20) | partly |
| G11 | Sampling / mode switching (LiteRace, Pacer-style) | — | changes detection; fails the zero-lost-races bar unless opt-in | **new**, out of the bar |

## 4. Contradictions and stale statements

| # | where | what |
|---|---|---|
| K1 | `runtime/README.md:20`, `runtime/ideas.md:11` vs `ideas.md:85`, `experiments.md:1278-1282` | RT-SYNC is listed as "parked 25 Sep", but it was censused, built, gated (29 Sep) and timed null on all five apps; RT-SYNC-LF (built 30 Sep, timed :1289) is missing from the runtime folder entirely |
| K2 | `runtime/README.md:12-14` | lists only ideas.md and als.md and says the range and slot notes "are not in this repository", yet range-hit-skip.md, slot-pref.md, sid-alias.md, dd-exact-ab.md and n1-atomic-a081.md are in the folder; sid-alias and DD-EXACT are not mentioned in README or runtime/ideas.md |
| K3 | `slot-pref.md:179-181` vs `runtime/README.md:55` | slot chains "≈ 5 % of server CPU, ≈ 21 % of user" (divided by the client's CPU) vs the 29 Sep correction "≈ 1.7 % of user, ≈ 1.0 % of CPU"; slot-pref.md was not corrected |
| K4 | `range-hit-skip.md:215-216`, `runtime/ideas.md:54`, `experiments.md:267` vs `runtime/README.md:59` | RANGE-UNIFORM "7-22 % of user, 1.6-5 % of server CPU" and "18 % of user / 4 % of server CPU" vs corrected "0.5-1.8 % of user, 0.3-1.1 % of CPU" |
| K5 | `runtime/README.md:24` (≈ 1.5 % of server user time) vs `experiments.md:269` (MemoryAccessRangeT 11.15 % of server user cycles) | range cost: census × warm per-cell cost vs perf; conservatism-review.md:§7.2 flagged it; README still quotes the census figure |
| K6 | `A3-runtime.md:52`, `:109` (slot functions 9.1 % of memcached) / `ideas.md:86` (slots 6 %) vs `runtime/README.md:55` (chains ≈ 1.7 % of server user) | perf shares vs TSC thread-time census differ ×4-5; not reconciled |
| K7 | `ideas.md:86` (deadlock detector ≈ 16-18 % of memcached server user cycles; DD-EXACT B1 +19-22 %) vs `hotspot-time.md:183-186` (dd 2.0 % of memcached on the record workload) | different workload and configuration; B1's gain was never re-measured on the record workload |
| K8 | `ideas.md:196` ("RT-STOCK-COST done: camera-ready runtime has upstream's instruction count") vs `optimization-results.md:80`, `:93` | upstream stock is still 1.047-1.049× faster than the root's stock on memcached ("compiler or runtime not separated"), 2-4 % on Redis; instruction parity is not time parity (cf. the TLS finding, phase-quiet-rt.md:408) |
| K9 | `optimization-results.md:113` vs `:90`, `:91` | "of the 1.29×, 1.085× is the runtime option of DynSTC-RT alone" vs "DynSTC-RT options on the stock binary 1.138" / "1.135×": two different quantities under the same wording |
| K10 | `experiments.md:1887` ((d) overwrite called "exact") vs `A3-runtime.md:176-177` ("zeroing redundant slots ... not a possible stock shadow", rejected) | the (d) lever is exact only at granule level (criterion G), whose scope is SUBS only (`soundness-fixes.md:416`) |
| K11 | `README.md:34` ("runtime/: nothing implemented without a decision") and `runtime/README.md` decisions vs the memory note runtime-track-separate (26 Oct: track opened as a second paper, implementation allowed in the tsan-rt lane) | the folder's README does not record the 26 Oct go |
| K12 | `ideas.md:9` | "Last update: 25 Sep 2026, 15:10" while the file has rows to 15 Oct |
| K13 | `soundness-fixes.md:426` (P-REPORT adopted 3 Oct: a second race on a reported cell may go unreported, for removal-mode DE) vs `upstream-pr-results.md:19, :160` ("the runtime fix closes the gap"; "the paper's runtime has this fix") | `_keep` landed 2 Oct (05efa6607c06) and covers covering *writes* only; P-REPORT still admits the read-cover case. The upstream text should say the fix closes the write case, or P-REPORT should name what it still covers |
| K16 | memory note camera-ready-root ("ONE compiler tsan-cr-8345a0396fa5 for all apps") vs `optimization-results.md:70, :93` | the adopted SQLite and memcached figures come from roots `tsan-lomark-913cae99c263` and `tsan-ecc-b54ce0196884`, descendants of 8345a whose runtimes add LO-MARK and EVCONF-CHECKED code that 8345a does not have |
| K17 | branch names in `/home/alexey/dev/llvm-project-focs-lab` | `atc26-camera-ready` (b66bdad3cfd3, 6 Oct; 14 squashed commits on 9f5d402cb36b) is not 8345a0396fa5 and adds R2, R3, PHASE-HIT-FIRST, self-off and tsan_fd.cpp; `experiment/camera-ready` is 8345a0396fa5 |
| K18 | `ideas.md:196` / A44 ("upstream's instruction count") vs the source at 8345a | by reading, a hit in `MemoryAccess` runs one more test-and-branch than upstream (`tsan_rtl_access.cpp:599`), and several stock paths load phase/LO flags (R25); no disassembly diff is recorded |
| K14 | `A3-runtime.md:181` ("Inlining relaxed atomics" rejected: moves signal delivery) vs `n1-atomic-a081.md:8-11` | superseded: N1-ATOMIC's hit path tests `pending_signals`; A3 is a frozen audit, but runtime/README's A3-derived summaries do not say so |
| K15 | `experiments.md:950` (DD-EXACT A MySQL 1.041, "resolved") vs `:1263` (0.996 over stock) and conservatism-review.md:472 (MySQL +3…+9 %) | the A gain did not recur over stock; the review's estimate predates it |

## 5. Code map (camera-ready tree and branches)

Repository `/home/alexey/dev/llvm-project-focs-lab`. Upstream base: **c609043dd009** (2024-05-06, upstream
"[clang-format] Don't allow comma in front of structural enum (#91056)"), found by `git merge-base 8345a0396fa5
origin/main`; local `main` points at it; no upstream merge since; no llvmorg tags in the clone. base..8345a0396fa5 is 504
commits, 109 of them touch `compiler-rt/lib/tsan`.

`git diff --stat c609043dd009..8345a0396fa5`: `compiler-rt/lib/tsan` 34 files +4430/−52; `compiler-rt/test/tsan` 126
files +8359/−16. Largest: new `tsan_phase.cpp` +1213, `tsan_filter.{cpp,h}` +584 (ReX, compiled out),
`tsan_rtl_access.cpp` +500, `tsan_rtl_amd64.S` +388, `tsan_interface_ann.cpp` +383, `tsan_rtl.cpp` +219,
`tsan_rtl.h` +208, `tsan_interface.inc` +168.

### Runtime features in 8345a0396fa5

| feature | idea | default | where (8345a) |
|---|---|---|---|
| `dynstc_rt`, `dynstc_single` | DynSTC-RT (R26) | off | `tsan_flags.inc:98`, `tsan_rtl.h:229`, `tsan_rtl_thread.cpp:132,185,391-401`, `tsan_rtl_access.cpp:41` |
| `__tsan_active_thread_count` | DynSTC (compiled) (C24) | on | `tsan_rtl.cpp:39`, `tsan_rtl_thread.cpp:128,358-391` |
| `*_miss2`, `MemoryAccessGivenTest`, `CheckInlineHitTestAbi` | N1 (C1) | entries off; ABI check on | `tsan_rtl_access.cpp:616,642,1158-1174` |
| `*_pm` | N1b (C2) | off | `tsan_rtl_amd64.S:233-339`, `tsan_interface.inc:130-175` |
| `FastState::single_`, mask test | N1-ST (c) (C3), stock path (R25) | layout + mask test on | `tsan_shadow.h:58-75`, `tsan_rtl.cpp:818` |
| `__tsan_check_batch*` | N2 (C5) | off | `tsan_rtl_amd64.S:361-561`, `tsan_rtl_access.cpp:1122` |
| `CheckInlineFuncEntryExitAbi` | FE-INL (C7) | on (start-up) | `tsan_rtl_access.cpp:699` |
| `*_miss2_fe` | FE-SINK (C8) | off | `tsan_rtl_access.cpp:1184-1209` |
| `__tsan_memcpy_nosrc`, `__tsan_memmove_nosrc` | MEMINTR (C11) | off | `tsan_interceptors_memintrinsics.cpp:53-74` |
| `__tsan_write*_keep`, `StoreAfterRace` | KEEP-ON-RACE (C12) | used by removal-mode DE | `tsan_defs.h:180`, `tsan_rtl_access.cpp:290,356,534`, `tsan_interface.inc:256-300` |
| `*_merged` | DE-MERGE (C22) | off | `tsan_interface.inc:312-344` |
| `*_range_all` | DE loop ranges (C23) | off | `tsan_interface.inc:244-248`, `tsan_rtl_access.cpp:1072` |
| `__tsan_holds_mutex`, `__tsan_lo_obj_*` | LO-OBJ-G / G-EA (C21) | gated; one load per lock/unlock/free | `tsan_interface_ann.cpp:469-816`, `tsan_rtl.cpp:66-114` |
| `tsan_phase.cpp`, `phase_guard*` | QUIET-THREADS (C15) | off; flag loads on stock paths | `tsan_phase.cpp`, `tsan_flags.inc:103-121`, `tsan_rtl.h:311-324` |
| hidden TLS `*_local` | RT-STOCK-COST (R25) | on | `tsan_rtl.cpp:59-114`, `tsan_rtl.h:404-422` (302d070bf406) |
| EA interceptor toggle via `ignore_interceptors` | EA (compiler) | start-up checks | `tsan_rtl.cpp:823-829` (A44 M2: fibers) |
| `__tsan_disable/enable` | quiet mode tests | on (no-ops) | `tsan_interface.inc:357-358` |
| ReX filter, eviction counters | research baselines | not compiled (CMake options off) | `tsan_filter.{h,cpp}`, `tsan_rtl_access.cpp:266`, `CMakeLists.txt:14,24` |
| reset ordering (`resetting` before the epoch moves) | fix | on | `tsan_rtl.cpp:205-207,298` (e442bc854efb, 2f5c69b2963d) |

Not in 8345a: N1-ATOMIC, RT-SYNC, RT-SYNC-LF, ALS, DD-EXACT A/B1, TSAN_ACCESS_STATS counters, SUBS, LIBCALL `_light`,
FE-LAZY, FE-PM, OWNER-RT, EVCONF-CHECKED markers, LO-MARK, carve/bound, R2/R3 phase entries, PHASE-HIT-FIRST.

### Runtime branches (tip, date, what they add over their merge-base with 8345a)

| branch | tip | date | holds | ancestor of 8345a? |
|---|---|---|---|---|
| experiment/als | c911b1f1ff67 | 10-10 | ALS; options `atomic_load_skip`, `_skip_cache`, `_skip_verify`, `_census`, `_mutant` (+312 in 8 files; 5 commits on 8345a) | no (child of 8345a) |
| experiment/rt-sync | 3da6710f0d43 | 09-29 | RT-SYNC, `skip_own_acquire` | no |
| experiment/rt-sync-lf | 61af8477c754 | 09-30 | RT-SYNC-LF, `skip_unchanged_acquire` (ea791e96cba6) | no |
| experiment/dd-exact | a2a7387e90aa | 09-30 | `deadlock_detector_large_table` | no |
| experiment/dd-exact-ab | 944fb9f4c502 | 10-01 | + `deadlock_detector_exact_prelock`, `print_deadlock_detector_stats` | no |
| experiment/dd-cheap | 40788cc8bbf7 | 09-29 | unconditional pre-lock skip (later flagged as 42dea40c3f32) | no |
| experiment/dd-census, measure/dd-census | 5e944dac269e, ab37a3e5a204 | 09-29, 10-06 | DD censuses | no |
| measure/sync-counters | c8f04181f253 | 09-25 | `sync_counters_path` | no |
| measure/slot-stats, measure/slot-cost | eece0eea7c77, c15e136e6771 | 09-28 | TSAN_ACCESS_STATS slot counters | no |
| measure/range-hit-astats, measure/range-uniform-astats, measure/range-census | 40be4af6535c, e03e517726c2, f4ef684205dd (incl. 8b5653e84bb0) | 09-28 … 10-01 | range and SID-ALIAS censuses | no |
| perf/access-stats | 1391bf330765 | 09-09 | origin of TSAN_ACCESS_STATS | no |
| experiment/subsumption-hit, experiment/subs-selective | 7c0a7c968e2e, add8aa7bfe27 | 09-29, 09-30 | SUBS, SUBS-SEL | no |
| experiment/n2-batch-checks, experiment/batch-checks | ad11b284a34f, 3b06462c8d7b | 09-24/25 | N2 (in 8345a already), prototypes | no |
| experiment/n1-atomic, experiment/n1-atomic-hot | fc762f401f92, e6b97f0ebc66 | 09-29, 10-01 | N1-ATOMIC | no |
| experiment/libcall-light | 6a1b9a0faf05 | 09-28 | LIBCALL `_light` | no |
| experiment/fe-pm, experiment/fe-lazy, experiment/fe-sink | c998c9c9a1a7, b6c5417665fd, d7e0ec71df33 | 09-28 … 10-02 | FE-PM, FE-LAZY, FE-SINK tests | no |
| experiment/owner-rt, experiment/evconf-checked-rt, experiment/fd-reuse | 2a14a2fbf2d5, 5fb6c13bcd60, 3a1bf6d1422f | 10-05 … 10-07 | OWNER-RT, EVCONF-CHECKED markers, descriptor model, PHASE-HIT-FIRST | no (children of 8345a) |
| experiment/phase-hooks-off, experiment/cr-phase-asm | 7ad8d3c7e668, ab32a9bea55e | 10-05, 10-06 | quiet-mode self-off, R2, R3, PHASE-HIT-FIRST | no |
| experiment/lo-mark, experiment/lo-mark-v2c | 718c7520052c, 36c805dd1906 | 10-08, 10-09 | LO-MARK, carve, bound | no (children of 8345a) |
| experiment/own-handoff | ddccf8375d9f | 09-29 | OWN-HANDOFF | no |
| fix/de-generation-guard | 0a4c719c7bd7 | 09-25 | old generation guard | no |
| experiment/phase-guard-rt, experiment/phase-quiet-rt, measure/quiet-census-v2 | 3fb7fd3017ed, f8061e814c0e, 1b1fc33f7591 | 10-02 … 10-04 | earlier quiet-mode forms, census | no |
| experiment/tls-hidden, experiment/camera-ready | 302d070bf406, 8345a0396fa5 | 10-05 | TLS fix; the camera-ready commit itself | yes |

192 of the 375 local branches carry lib/tsan commits not patch-equivalent to any commit in 8345a (334 commits). Full
tables:
- runtime/code/branch-newrt.tsv
- runtime/code/branches-all.tsv
- runtime/code/notinR.tsv
