# TSan instrumentation optimizations: results by category

State as of 27 Sep 2026, 16:40 (focs time). Preliminary: most cells come from one to three sessions at N = 3. Resolution is
about 2-4 % (MySQL about 5 %); a gain of |x| ≲ 1-2 % is noise.

**How to read every table**
- A number is the speed gain in percent: (speed ratio − 1) × 100, positive = faster.
- Unless a row says otherwise, the reference is **P1-v3**, the corrected base (defined below). The totals in table 3 are against the
  **shipped** artifact.
- **f** = focs (Intel Xeon w9-3495X, 112 threads), **a** = apollo (2 × AMD EPYC 9115). "f −8.9 · a +4.2" means −8.9 % on focs
  and +4.2 % on apollo. A cell with one letter was measured on that machine only.
- "?" = sessions disagree (unresolved); "—" = not measured.
- Every configuration in tables 1-3 is **race-preserving**: it loses no race that stock TSan reports (zero lost races, checked by
  IR tests, check-tsan, reproducers with controls and an independent audit), under the premises stated in the project.

**The base, P1-v3.** The shipped configuration (EA, LO, STC, SWMR, and DE with loop peeling), with DE made exact ("verified
removal": a check is removed only when an inline copy of TSan's hit test proves that the check would hit), plus every soundness
fix found after shipping. P1 over shipped: SQLite +0.3, memcached +1.4, Redis −2.9, MySQL 0…+3, FFmpeg −0.3.

## Table 1. Optimizations that gain somewhere

| optimization | SQLite | memcached | Redis | MySQL | FFmpeg |
|---|---|---|---|---|---|
| FE-INL | a −0.7 | a +0.8 | f **+4.8** | f −8.9 · a **+4.2** | f 0.0 |
| FE + VWIDE | a 0.0 | a −0.3 | f **+5.2** | f −8.6 · a **+4.5** | f +2.0 |
| FE + VWIDE-loops | a +0.1 | a +0.6 | f **+6.4** | f −7.6 · a **+4.5** | f +1.8 |
| N1 | f −2.1 ? · a −0.8 | f −2.7 · a +0.3 | f **+3.7** | f −1.4 · a −1.7 | f **+5.0** |
| N1-L | f 0.0 ? · a +0.3 | f ? · a −0.9 | f −2.5 | a −0.1 | f **+3.0** |
| DynSTC-RT | f +0.8 ? · a +0.4 | f +2.6 ? · a −0.3 | f −2.2 | a +0.1 | f **+12.0** |
| N1-ST front ¹ | f 0.0 ? · a +0.2 | f −0.8 · a −0.2 | f −1.4 | f −1.5 · a −1.8 | f **+16.9** |
| N1-ST miss ¹ | f −4.3 ? · a +2.0 | f −2.8 · a +0.1 | f +2.1 | f −0.4 · a −2.6 | f **+9.8** |
| exact DE package ² | a +0.8 | f **+3.6** | f −2.6 | f **+3…+6** | f +0.5 |

¹ Measured on top of N1 with DynSTC-RT; on FFmpeg, on top of DynSTC-RT + FE + N1; on Redis, without DynSTC-RT.
² Measured over the shipped artifact (tier A), not over P1-v3.

**What these optimizations are**
- **FE-INL** (inline function entry/exit): the push and pop of TSan's shadow call stack are inlined instead of calling
  `__tsan_func_entry/exit`. The trace events are identical. It gains on Redis and on MySQL on AMD. On MySQL on Intel it loses: FE
  runs up to 8 % fewer instructions per transaction but no fewer cycles there (the +59 % code size raises i-cache/iTLB stalls), and the
  debug-build MySQL is lock-bound, so longer transactions hold locks longer. The processor decides the sign, contention the size.
- **VWIDE** (idea 7, "cheap run-time DE checks"): exact removal also at sites that are not dominated by a covering check but usually
  hit in practice; an inline hit test verifies the cover at run time. **VWIDE-loops**: only inside loops. Alone they gain nothing
  (table 2); they are listed here in combination with FE.
- **N1**: TSan's fast-path hit test ("is this access already recorded in shadow?") is inlined at every access; the runtime is called
  only on a miss. **N1-L**: the same, only in small loops.
- **DynSTC-RT**: a run-time single-thread mode. While the process has one live thread (none created yet, or all others joined), the
  runtime records nothing; the create/join edges order everything else. Its whole gain is FFmpeg's single-threaded stream copy.
- **N1-ST**: N1's inline test is skipped while the thread is in that single-thread mode (the test would always miss there). **front**:
  the flag is read before the test (largest gain on single-threaded code, 1-3 % tax on multi-threaded code). **miss**: the flag is read
  only on a miss (no tax on hits, about half the gain).
- **exact DE package**: DE's merging of adjacent checks and loop ranges, made exact under verified removal.

## Table 2. Optimizations that gain nothing (or lose)

| optimization | SQLite | memcached | Redis | MySQL | FFmpeg | status |
|---|---|---|---|---|---|---|
| VWIDE (alone) | a −0.6 | a +0.6 | f −0.5 | f −0.3 · a −0.3 | f +0.8 | noise; no effect together with N1 |
| VWIDE-loops (alone) | a +0.2 | a 0.0 | f −0.9 | f −1.8 | f +1.3 | noise |
| FE-INL exit-max=1 | a = FE | — | — | f −5.2 | — | variant, dropped |
| N1b | f −4.3 · a +0.8 | f +3.0 ? · a −2.7 | f −1.3 | a −3.7 | f −0.8 | dropped |
| N1-S | a +0.4 | a +0.1 | f −2.2 | f −2.0 | f +0.5 | closed |
| WP | a −1.5 | a +0.1 | f +0.6 | ≈ 0 (static) | f −0.3 | no gain |
| N2 ² | a −1.9 | f −0.4 | f −1.7 | dropped | f −0.1 | reference only |
| SUBS | a −18.2 | a −10.9 | f −2.0 | a −8.0 | f −14.7 | closed (27 Sep) |
| loop guard ² (not race-preserving) | +1.0 | −0.7 | +2.1 | f −5.2 | — | excluded |
| all inexact T10 ² (not race-preserving) | −1.2 | +1.7 | +0.9 | f −6.6 | +20.5 | excluded |

**What these optimizations are**
- **FE-INL exit-max=1**: FE-INL with at most one inlined exit per function, a variant for C++ unwinding.
- **N1b**: the runtime call is kept but goes through cheaper `preserve_most` entry points (fewer saved registers).
- **N1-S** (idea 6): N1 only at statically hot sites, with a budget of 2 per function, no profile. It misses the hot sites of large
  functions.
- **WP**: whole-program summaries for the static analyses (closed world, `-tsan-external-symbols`).
- **N2**: several checks batched into one runtime call.
- **SUBS**: a same-thread, same-epoch shadow record that covers the access's bytes with an equal or stronger kind counts as a hit;
  with an absorbing store and exact merging. The extra run-time test on every miss costs more than it saves.
- **loop guard / T10**: the inexact DE variants (checks removed by coverage without run-time verification). They can lose races
  under TSan's bounded shadow and are excluded; shown for reference only.

## Table 3. Cumulative results: the best combination per app and one universal combination

| configuration | SQLite | memcached | Redis | MySQL | FFmpeg |
|---|---|---|---|---|---|
| best per app, measured together (over P1-v3) | nothing gains | a +0.6 (FE + VWIDE-loops); f nothing | f **+9.1** (FE + N1; 2 sessions) | a **+4.5** (FE + VWIDE-loops); f nothing | f **+33.5** (DynSTC-RT + FE + N1 + N1-ST front; 3 sessions); +14.2 without N1-ST |
| universal U1 = N1 + N1-ST miss + DynSTC-RT (over P1-v3) | f −4.6 ? · a −1.6 | f −5.0 | f +4.7 | a −4.3; f pending | f +25.1 |
| universal U2 = U1 + FE (over P1-v3) | f −4.4 ? · a −1.1 | f −4.7 | f +7.5 | a +1.9; f pending | f +25.3 |
| **total over shipped, best per app** | ≈ +0.3 | f ≈ +1.4 · a ≈ +2 | **≈ +5.9** | f 0…+3 · a ≈ +4.5…+7.5 | **≈ +33** |
| ceiling: O1-all / O2-eraser (oracles, not optimizations) | +15 / +52 | +4 / +9 | +18 / +28 | −1 / +3 | +31 / +41 |

**How to read this table**
- **Best per app**: the best race-preserving combination for that app, with all its optimizations enabled in one binary and measured
  in one session (not a product of single gains).
- **Universal U1/U2**: one configuration applied to every app. It gives up a little on Redis and about 8 points on FFmpeg against the
  best per app, and it costs SQLite and memcached on focs, where N1's inlined code is slower.
- **Total over shipped**: the best combination including P1's own effect, i.e. the whole speedup a user of the shipped artifact
  would see.
- **Ceilings**: profile oracles that skip every check on memory touched by one thread (O1-all) or also on memory Eraser would exempt by
  a common lock (O2-eraser). They are unsound upper bounds on what any analysis of that kind could reach, not optimizations.
- **In progress (batch 11, tonight)**: the **optimistic** configuration (the best arm per subtest, chosen from earlier sessions and
  read in a fresh session; pre-registered in `batch11-preregistration.md`) and the **realistic** one (one arm with the best geomean
  over all apps), on focs for all apps and on apollo for SQLite, memcached and MySQL. Arms: base, U3 = FE + VWIDE-loops, U4 = FE + N1,
  U2, U5 = FE + N1 + N1-ST front.

## Table 4. Ideas not yet timed, or closed without timing

| idea | what it is | status | ceiling or expected gain |
|---|---|---|---|
| EA-P5/P6/P7 (soundness) | today's EA loses races when a pointer to a local is published through a pipe (write/read), through `%p` text, or through the generic `__atomic_load` libcall | reproduced (stock 10/10, EA 0/10 each); fix designed, paused pending a decision | correctness, not speed |
| LO-OBJ | lock ownership relative to an object: fields of objects owned by one SQLite BtShared (pages, WAL, Pager), accessed while its mutex is held | design and cheap checks done; needs ownership annotations taken from SQLite's own `mutex_held` assertions | SQLite ≈ ×1.17-1.21 (upper estimate); memcached ≤ +0.2; MySQL ≈ 0 |
| EA-SLOT (idea 8a) | a value loaded from a stack slot and passed on escapes as the slot's contents, not the slot (memcached's `tokens` array) | repaired design v2 found unsound by audit (3 new counterexamples, fixes C11-C13); census branch ready | ≈ 3 % of memcached's checks |
| MEMINTR (idea 8b) | memcpy/memmove from a constant or uncaptured-local source: check only the destination | implemented; audit found and the author fixed a declaration/interposition hole; re-audit running; gate tonight | < 1 % of checks |
| CLONE-ESC (idea 9) | clone functions with hot pointer arguments into escaping and non-escaping versions; choose the clone at run time | ceiling measured by profile; timed oracle arm queued | ≤ 7 % FFmpeg/Redis, ≤ 3 % SQLite, ≤ 1 % memcached, 0 MySQL |
| N1-ST (b), (c) | hoist the single-mode flag read per call-free run; or encode single mode in the fast state TSan already loads | not built | would halve or remove N1-ST front's 1-3 % multi-threaded tax |
| sound loop ranges | one range check per loop placed after the loop or per chunk (the pre-loop form loses races and was rejected) | candidate | FFmpeg swscale loops ≈ 17 % of O1 mass |
| SWMR-H | a location written only before its publication is exempt afterwards | parked (audit found 14 lost-race paths; needs a premise on indirect calls) | U1 ceiling: memcached +0.2, Redis 0, SQLite 0 |
| LO-F | lock ownership per field | parked | ≈ 0.5 % of memcached's checks |
| LO-W | recognise mutexes LO cannot identify today | closed by ceiling | Eraser ceiling ≤ 1.4 % |
| EA-7 | refine "arguments of address-taken/external functions escape" | open, static count only | +5.5 k static sites |
| EA-WP | EA with whole-program summaries | census: nothing on 3 apps | ≈ 0 |
| EA-TL / EA-HEAP | thread-local memory / "own heap" statically | no-go (the objects are reachable from globals) | — |
| SWMR-G | write-once-then-read-only memory | no static route | shared read-only memory 4-14 % of checks |
| DE-2 | merge adjacent same-kind fields into one ≤ 8-byte check | not built | ≤ 1.1 % SQLite |
| DE-5 | cycle scan only over cycles that avoid the cover | census | ≈ 1 % of checks |
| DE-1, DE-AA, DE-4, DE-6..9 | more same-address proofs, stronger alias analysis, cover at an offset, call/fence/allocation/constant-data judgments | census | 0-0.2 % |
| DE-10 / DE-AV | availability instead of strict dominance / a dynamic loop guard for any repeated address | DE-10 covered by VWIDE; DE-AV idea only | — |
| N6 | subsumption-aware shadow eviction | soundness note, open | — |
| STC-1..4, DYN-1 | more single-threaded-context rules; drop DynSTC guards where more threads certainly exist | oracle bound: nothing hot | ≈ 0 |
| LO-B1/B2 | private mutexes immune to unknown unlocks; callback releases | unsound, closed | — |
| RT-SYNC/RANGE/ALLOC/SLOT | runtime-only speedups (sync path, range checks, allocator, slot machinery) | parked: outside the paper's topic | memcached slot machinery ≈ 9 % of cycles |

**Terms used in this table**
- **Ceiling**: an upper bound from a profile oracle or a census of executed checks, not a timed optimization.
- **Census**: a static or profile count of the checks a rule would remove, weighted by how often they run.
- **Parked / closed**: parked = set aside with its data kept; closed = measured or argued to have no worthwhile gain, or unsound.
- **Audit**: an independent read-only review against the zero-lost-races rule, before anything is timed.
