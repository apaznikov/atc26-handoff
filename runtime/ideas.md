# Runtime-only ideas

Ideas that change only TSan's runtime (compiler-rt), not the instrumentation pass. Kept separate from the paper's
track by Alexey's decisions: 25 Sep 19:25, "pure runtime; the paper is about static analysis and instrumentation, not
a new runtime — revisit later"; 28 Sep, "everything about the runtime goes into a separate folder". Overview and
estimates: `runtime/README.md`. Russian: `runtime/ideas.ru.md`. Moved here from `ideas.md` on 28 Sep 2026; the rows
are unchanged.

### RT-SYNC

- **What:** Exact sync-path speedups in the runtime: write only changed clock lines, skip no-op acquires (per-SyncVar version), one SyncVar lookup per lock op (audit A3)
- **Status:** **parked (Alexey 25 Sep 19:25: pure runtime; the paper is about static analysis and instrumentation, not a new runtime — revisit later)** — sync = 33 % of memcached's and ~55 % of MySQL's cycles; estimate MySQL 5-15 %, memcached 2-6 %; counters branch `measure/sync-counters` c8f04181f253 kept
- **Owner:** tsan-dev-2

### ALS (10 Oct 2026): atomic load skip

- **What:** A special case of RT-SYNC's "skip no-op acquires (per-SyncVar version)". Each sync object carries a version word naming the present state of its clock (0 = being written, 1 = empty, otherwise unique and never reused). An acquire-order atomic load that sees "empty", or a version this thread has already joined (64-entry per-thread cache), and still owns its slot, skips the slot lock, the sync's reader lock and the vector-clock join; the access check itself stays. Clocks and reports equal stock's on a legal reordering
- **Status:** **parked (Alexey 10 Oct: the runtime is not to be touched; the paper is about static analysis and instrumentation)**. It was implemented, audited, frozen as a root and put into Chromium legs on 10 Oct before that ruling, in breach of the 25/28 Sep rule; those legs are not in the tables. Nothing further is built
- **Owner:** tsan-dev-2
- **Evidence:** audit A84 with three deltas (`../audit-a84-atomic-load-skip.md`): SOUND after the slot-ownership check; micro-benchmark 52 → 16 ns per with-sync load, eight threads on one atomic 2,000 → 110 ns; Chromium census: 39-44 % of atomic loads take the sync path, 86-90 % of those are eligible under default options (94-97 % with the harness's 2 s reset); timing on one binary, flag on over off: 1.02 over 49 blink stories on AMD and on Intel, parser 1.13. Branch `experiment/als` c911b1f1ff67, root `tsan-als-c911b1f1ff67`. Details: `als.md`
- **Next:** if the runtime track is reopened: memcached and Redis (atomics-heavy); the SyncVar-pointer cache with an incarnation id (est. 2-4 % of samples); x12 and preservation with the flag on at the frozen root

### RT-RANGE

- **What:** Range checks: SIMD over 2-4 granules, no trace event when all hit (audit A3)
- **Status:** **parked (Alexey 25 Sep 19:25: pure runtime; the paper is about static analysis and instrumentation, not a new runtime — revisit later)** — ranges = 10-15 % of cycles; R1 (SIMD, bitwise identical) ≈ 2-4 %, R2 (no trace event on all-hit) under criterion (G)

### RT-ALLOC

- **What:** malloc/free in the runtime: faster freed/reset shadow fill; cache the allocation stack id by call stack (StackDepot::Put) — Redis ≈ 11 % of cycles
- **Status:** **parked (Alexey 25 Sep 19:25: pure runtime; the paper is about static analysis and instrumentation, not a new runtime — revisit later)**
- **Next:** profile first if revived

### RT-SLOT

- **What:** Slot machinery (memcached 9.1 % of cycles in SlotLock/Unlock/AttachAndLock); stop preempting live slot owners; lossless eviction victims first (detection, not speed)
- **Status:** **parked (Alexey 25 Sep 19:25: pure runtime; the paper is about static analysis and instrumentation, not a new runtime — revisit later)**

### RANGE-HIT-SKIP

- **What:** Skip MemoryAccessRange's trace event (and the per-cell work) when every cell of the range already holds the same access, as the single-access path skips its event on a hit
- **Status:** **no-go** (28 Sep 10:50: census all-hit share FFmpeg 81 %, SQLite 73 %, Redis 50 %, memcached 11 % → bound ≤ 0.53 % realistic, 1.1 % conservative on SQLite only; correct but not worth it; RANGE-VEC ≤ 0.9 %, not opened) — earlier: open (07:55); argument holds under P-EV + P5 (stock already lazy-traces MemoryAccess16/Unaligned via `traced`; `range-hit-skip.md`); census staged; prior ceiling < 1 % except memcached-user 3.3 % / SQLite 1.3 %. Companion candidate RANGE-VEC (a cheaper hit walk, e.g. 2 granules per AVX2 compare): a hit cell costs ~0.7-1 ns, so on memcached up to ~7 % of user / ~2 % of server CPU if cells hit; decided by ns/cell (mb) × hit share (census)
- **Evidence:** LIBCALL's cost model: range work dominates compare calls; memcached does 4.7 G range calls / 360 GB of range bytes per run
- **Next:** tsan-dev-3: soundness argument against the code (stack restoration, eviction, reads vs writes, partial granules) + census of all-hit range calls/bytes per app; no implementation before both

### SLOT-PREF

- **What:** FindSlotAndLock prefers an unowned, non-exhausted slot (free list or bounded scan) and preempts a live thread only when none is free; stock takes the queue's front slot even when it is owned
- **Status:** **open** (28 Sep 10:34); 10:50 (tsan-dev-3): the pick is already free-first (free slots stay at the queue front); preemptions are chains that start only when no free non-exhausted slot is left and end at the reset → the lever is the epoch budget (1.9 G increments per memcached run, release-driven): census of free slots at preemption, rdtsc costs, plain-empty epochs (bound for an epoch-eliding release)
- **Evidence:** slot counts per run: memcached 13.96 M preemptions (≈ every attach) and 456 DoResets (each wipes all shadow); MySQL 2.6-2.8 M, ~200; FFmpeg ~150 k; Redis 0. Every preemption is a stock race-loss window, and each step takes the global slot_mtx
- **Next:** tsan-dev-3: confirm the cascade from the code, cycle cost (attach/relock/reset, slot_mtx waits) as a share of user CPU, design + soundness under P5 (expected to reduce loss windows), counting run with it; then audit, gate, timing on memcached/MySQL

### RANGE-UNIFORM

- **What:** On a range miss, when consecutive cells hold identical shadow (modulo in-cell address bits), race-check one cell and store the rest with wide stores
- **Status:** **CLOSED 29 Sep** (upper bound ≈ 2 % of memcached's server user cycles, below resolution; tsan-improve 23:3x)
- **Evidence:** memcached: 45.3 G of 46.5 G range cells MISS per run (97.6 %; fresh buffers), ~1.25 ns each ≈ 18 % of user / 4 % of server CPU; FFmpeg 12.6 G (≈ 1.8 %)
- **Next:** none. Census (28 Sep): 76.7 % of memcached's missed cells equal the previous cell. Perf (29 Sep): MemoryAccessRangeT = 11.15 % of memcached's server user cycles, ≈ 21 cycles per missed cell, in the per-cell SIMD compare. Microbenchmark (29 Sep 23:17, /extra/alexey/wt-dev2-r/rangemb, control EXACT): cold fresh shadow at u = 0.77 saves 23 % per cell (8.99 -> 6.91 TSC cycles), foreign cells +2 %; below the 30 % bar. The benchmark's cold cell costs 9 cycles against memcached's ~21: the rest is scattered ranges and TLB/page-level misses, which a wide store would still pay (it touches every shadow page), so a page-level variant does not help either.


### RANGE-OWN-FASTPATH (6 Oct): bulk store when every slot in the range carries this thread's sid
- **What:** For a range access whose shadow slots are all empty or own-sid, skip per-cell CheckRaces and bulk-store the current access.
- **Status:** **NOT PURSUED 6 Oct** (decided 6 Oct). It cannot avoid touching the shadow; the closed wide-store idea above bounds the per-cell saving at ≈ 2 % of memcached's server cycles; the census that finished anyway (range-hit-skip.md §9) puts it at 0.3 % (only 7.8-10.2 % of missed cells are own-sid), < 0.1 % on zmr's line, 0.2 % on Redis. The memset mass is taken by EVCONF-RANGES's static ownership instead.

### DD-STATIC (6 Oct): skip the lock-order detector's bookkeeping on lock classes no static cycle can reach
- **Status:** DD-STATIC: not pursued, out of the paper's scope (6 Oct). Lock-order detection is a different task from race detection.
