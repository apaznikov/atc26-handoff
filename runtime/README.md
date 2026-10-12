# TSan runtime track: ideas, estimates, censuses

Everything that changes only TSan's runtime (compiler-rt), not the instrumentation pass, lives in this folder. It is kept separate from the paper's track, which is about static analysis and instrumentation.

**Decisions:**
- **25 Sep 19:25:** RT-SYNC, RT-RANGE, RT-ALLOC and RT-SLOT parked as pure runtime: the paper is about static analysis and instrumentation, not a new runtime; to be revisited later.
- **28 Sep:** the range and slot questions came back through the cost analyses of LIBCALL and FE-SINK. They were taken only as censuses (estimates, no implementation). It was then decided to put everything about the runtime into this separate folder.
- **29 Sep ~12:30:** decided to try the runtime ideas on memcached too. Order: RT-SYNC no-op acquire skip → RANGE-UNIFORM; slot-chain breaking waits for the decision on P5. Exact against stock; results in their own "runtime" rows.

**Files here:**
- `ideas.md`: one entry per idea, with status, evidence and next step;
- `als.md`: the atomic load skip (ALS), with its mechanism, audit and numbers.

The detailed notes on the range ideas (RANGE-HIT-SKIP, RANGE-VEC, RANGE-UNIFORM: cost model and census results) and on the slot preemption chains (epoch budget, epoch-elision analysis) are not in this repository; their results are summarised below.

## Overview

| idea | what it is | status | estimate (measured unless marked) | where |
|---|---|---|---|---|
| **RT-SYNC** | sync path: write only changed clock lines, skip no-op acquires (SyncVar version), one SyncVar lookup per lock op | ⏸️ parked 25 Sep | sync = 33 % of memcached's cycles, ~55 % of MySQL's (audit A3 profile) | `ideas.md` |
| **ALS** (atomic load skip) | an instance of RT-SYNC's "skip no-op acquires": an acquire-order atomic load whose sync object's clock is empty, or is a clock state this thread has already joined (per-SyncVar version word), skips the slot lock, the reader lock and the join | ⏸️ parked 10 Oct (decided: the runtime is not to be touched). Built and audited before that, against the rule; kept as an idea only | Chromium: 21-32 % of browser CPU on that path; 86-90 % of such loads eligible under default options; 52 → 16 ns per load; +2 % over 49 blink stories on both hosts (parser +13 %), no code or memory cost | `als.md` |
| **RANGE-HIT-SKIP** | skip a range's trace event when every cell already holds the access (stock already does this for 16-byte and unaligned accesses) | ❌ closed 28 Sep by census | all-hit share of range calls: FFmpeg 81 %, SQLite 73 %, Redis 50 %, memcached 11 %; bound ≤ 0.53 % (≤ 1.1 % conservative, SQLite only) | `ideas.md` |
| **RANGE-VEC** | vectorised walk of hit cells (two granules per compare) | ❌ not opened | a hit cell costs 0.47 ns; bound ≤ 0.9 % (SQLite), < 0.4 % elsewhere | `ideas.md` (under RANGE-HIT-SKIP) |
| **RANGE-UNIFORM** | on a range miss, when consecutive cells hold identical shadow, race-check one cell and store the rest wide | ❌ closed 29 Sep (census 28 Sep, window 19; upper bound ≈ 2 % of memcached's server user cycles, below resolution) | memcached: 45.3 of 46.5 G range cells miss per run (97.6 %), ~1.25 ns each, ≈ 1.5 % of the server's user time (corrected 29 Sep: earlier shares divided by the load client's CPU). Exact against the stock walk with no premise, but only if every cell is still loaded; whether the cost is logic or cold shadow was open at the census (the measurements of 29 Sep are in `ideas.md`) | `ideas.md` |
| **RT-ALLOC** | faster freed/reset shadow fill; cache the allocation stack id by call stack | ⏸️ parked 25 Sep | Redis ≈ 11 % of cycles | `ideas.md` |
| **RT-SLOT / epoch budget** | stock TSan's slot preemption chains and global resets | 🔎 cost census 28 Sep (window 19) | Per run: memcached 14 M preemptions and 456 DoResets (each stops the world and wipes all shadow), MySQL 2.6-2.8 M and ~200, FFmpeg ~150 k and 16, SQLite 7-11 k and 5, Redis 0. The slot pick is already free-first (SLOT-PREF has little to take, except maybe on MySQL). Resets are release-driven (≈ 1.9 G epoch increments per memcached run). Epoch elision at releases is unsound (every release records its own sync access). Earlier profile: slot machinery 9.1 % of memcached's cycles | `ideas.md` (RT-SLOT, SLOT-PREF) |

## Runtime parts of compiler levers (tracked in the main tables)

These are runtime entries that serve instrumentation levers. They are listed here for reference only; their results are in `../optimization-results.md`, and those of N1b, N2 and LIBCALL-INLINE only in the full ledger (`../optimization-results.md` at commit 9126669).
- **DynSTC-RT:** the run-time single-thread mode; FFmpeg +12 %.
- **N1/N1b miss entries:** `__tsan_*_miss2` and the preserve_most assembly entries.
- **N2:** the batch entries; dropped.
- **LIBCALL-INLINE:** the light compare entries. Closed: they save 3-6 ns per call, and the range checks both paths pay dominate.
- **FE-SINK:** the `miss2_fe` entries.
- **MEMINTR:** the `_nosrc` entries.

## Censuses and measurements (raw data)

The raw data (summary files of 28 Sep and per-run counter files) are kept on the Intel host, not in this repository.

- **Slot counts:** a summary of 28 Sep (per app and tree, counters summed over each run's counter files). Root `tsan-fesink2-slotstats-eece0eea7c77`; trees c6st (stock) and c6p1 (paper configuration).
- **Range census:** a summary of 28 Sep (all-hit share of traced range calls: FFmpeg 81 %, SQLite 73 %, Redis 50 %, memcached 11 %). Root `tsan-rangehit-astats-40be4af6535c`, tree c8rh.
- **Microbenchmarks:**
  - per-call cost of the stock and light compare entries;
  - range trace event 1.69 ns;
  - hit cell 0.473 ns, missed cell ~1.25 ns (quiet, CPU 54, 4.3 GHz).
- **Window 19b (28 Sep 14:27-15:25, untimed, one run per app per tree):**
  - **Slot cost:** root `tsan-fesink2-slotcost-c15e136e6771`, trees c8st (stock) and c8p1 (paper configuration). Slot time summed over threads (find + attach + re-lock): memcached ~144 s per ~171 s run (12.2-12.5 M preemptions); MySQL ~174-181 s per ~421 s run (2.3-2.4 M preemptions; lock waits ~56 s); FFmpeg ~2 s; SQLite < 0.1 s.
  - **Uniform cells:** root `tsan-rangeuni-astats-e03e517726c2`, tree c8ru (P1-v3). Share of missed range cells that are uniform: FFmpeg 93.5 %, memcached 76.7 %, SQLite 63.4 %, Redis 2.1 %.
  - **memcached server page faults per run:** stock (c8st) minflt 1 992 234, majflt 1; paper (c8p1) minflt 1 991 338, majflt 0 (read from `/proc/<server>/stat` just before the server stops).

## Readout of 28 Sep (window 19b; estimates only)

- **Slot chains:** memcached ≈ 66 s per run (re-lock 53.4 s at 4.4 µs per step, half of it waiting for the global slot_mtx, plus contended SlotLock waits 12.6 s) ≈ 1.7 % of the server's user time, ≈ 1.0 % of its CPU (corrected 29 Sep from 21 % / 5 %, which divided by the load client's CPU); resets ≤ 0.5 %. MySQL ≈ 125 s (share to be re-checked: its time also wraps a client script) (resets cost 129 ms each). FFmpeg ≈ 0.3 %, SQLite 0.
- **SLOT-PREF closed, measured:** only 1.5 % of memcached's sampled preemptions found a free non-exhausted slot.
- **Epoch budget:** 12.2 % of memcached's 1.89 G epoch increments close a truly empty epoch (likely statically initialised mutexes); eliding them would cut preemptions and resets ≈ 12 % (~8 s of the 66).
- **Candidate lever: end the chains** with an early reset; its detection effect (more shadow wipes vs fewer false happens-before preemptions) needs a preservation run first.
- **RANGE-UNIFORM:** missed range cells equal to the previous one: memcached 76.7 % (34.6 of 45.1 G), FFmpeg 93.5 %, SQLite 63.4 %, Redis 2.1 %. Estimate: memcached 0.3-1.1 % of server CPU (0.5-1.8 % of its user time; corrected 29 Sep from 1.6-5 % / 7-22 %), FFmpeg 0.8-2.7 %, SQLite ≤ 0.7 %; the per-cell saving needs a cross-thread miss microbenchmark.

## Link to the paper's track

Premise **P5** is the one runtime fact the paper's soundness depends on. It says four things are arbitrary: the evicted slot, the recycled trace part, the slot an attach takes, and the moment of a re-lock.
- The slot counts above show why it matters: every preemption is a window in which stock TSan already loses races, and every DoReset wipes all race history.
- P5 was adopted on 1 Oct (`../soundness-fixes.md`, section 6a).
