# Hotspots: where the shipped configurations still pay, and why (4 Oct 2026)

**Status: current reference (4 Oct 2026).**
- Counting, explanation and the joins are the hotspots census of 3-4 Oct; the time attribution is the perf split of
  3-4 Oct. Counting and profiling only; nothing here changes a configuration.
- This report replaces the per-topic hotspot notes as the current reference. History: `missed-census.md` (30 Sep,
  default configuration), `hotspot-blockers.md` (1-2 Oct: Redis `server`, SQLite shared cache, memcached
  connections, MySQL Release), `ideas.md` (the lever ledger).
- Russian: `hotspots.ru.md` (not committed).

## Summary

- **Where the time goes.** Plain checks lead on SQLite (50.6 % of cycles), MySQL (40.6 %) and FFmpeg's mjpeg
  (58.4 %). Ranges and interceptors lead on Redis's main thread (36.4 %) and memcached (31.3 %), and memcached also
  pays 24.8 % in mutexes. Runtime internals and the deadlock detector are ≤ 2 % everywhere.
- **Plain checks are flat; ranges are not.** No plain-check location exceeds 5.4 % of an app's cycles (SQLite and
  MySQL ≤ 1 %). Two range sites dominate:
  - memcached's `memset(resp, 0, sizeof(*resp))` in resp_allocate: **16.6 %** of all cycles;
  - Redis's reply-list `memcpy` in _addReplyProtoToList: **14.1 %** of the main thread.
- **Hit rates differ by an order of magnitude.** memcached's plain checks hit only 12 %, since every request's
  mutexes start new epochs. MySQL hits 55 %, SQLite 71 % and Redis's main thread 97 %. Where checks miss, each
  removal saves a slow path.
- **What still blocks the plain checks.**
  - EA's pointer-argument refusal is the largest PHANTOM class: 17-45 % of plain checks. Its static routes are
    closed by count (top-down facts ≤ 0.14 %); its run-time routes are quiet mode (Redis, in work) and own stack
    (parked).
  - The non-global T2 objects that LO and SWMR cannot judge (7-43 %) are LO-OBJ-G's ground (SQLite).
  - Genuinely shared memory (S) is the largest REAL class: 59 % on memcached, 63 % on MySQL.
- **New candidates, by ceiling.**
  - EVCONF-RANGES: memcached ≤ 16.6 %. Measured 4 Oct: +13.9 % over cmb on Intel, equal to the oracle (1.141), and
    +10.3 % on AMD; it goes into memcached's camera-ready configuration.
  - Quiet mode's range skip: Redis main ≈ 19 %; already written.
  - LO-OBJ-G on `db->mutex` objects: SQLite create_drop_index_1 ≈ 5.5 % of its checks.
  - LO-OBJ-RANGES: SQLite ≤ 5.7 % of cycles; out (4 Oct): it reaches neither site (candidate table).
  - EVCONF-ARGS: memcached ≤ 5.4 %; its oracle reads 1.082 over cmf (Intel, 4 Oct), and the design is in audit (A50).
- **Corrections.** MEMINTR-INLINE's census (3 Oct) called memcached's response memset cold; it is the hottest range
  of all (memintrin-inline-design.md §6, corrected).

## Question

For each app's camera-ready best configuration (root `tsan-integcr-1738fdee35e7`, cr1738.spec):
1. where the remaining overhead over native goes;
2. which instrumented locations cost most, on which object, and why each is still checked;
3. which classes of blocker remain, with their mass, marked closed, parked or new.

| app | best configuration (cr1738.spec) | workload (record) |
|---|---|---|
| memcached | AllOpt + WP summaries, `-tsan-de-verified`, EVCONF spec 3, SWMR thread roots, local-globals summaries (V4) | memtier, pipeline 32, multi-key get 32, 100-byte prefix, 8 threads |
| SQLite | AllOpt, `-tsan-de-verified`, LO-OBJ-G spec v7 | create_drop_index_1 + stress2, 60 s |
| Redis | AllOpt, `-tsan-de-verified`, FE-INL, N1 | 12 I/O threads; LRANGE_100/300/500/600, ZADD, ZPOPMIN, MSET |
| MySQL | AllOpt, `-tsan-de-verified`, FE-INL, RelWithDebInfo | sysbench write set, 24 connections; timed phase only |
| FFmpeg | AllOpt, `-tsan-de-verified`, N1, N1-ST (front) + DynSTC-RT | the reference clip through h264, h265, mjpeg, copy |

## Method

- **Time split (item 1):** a perf attribution of each best configuration (cycles:u with LBR call chains, 3 runs), per
  runtime symbol, mapped to: plain-access checks (inline hit test, runtime miss path), function entry/exit, range checks and
  interceptors, atomics and sync, runtime internals (slot preemption, global resets, trace), deadlock detector.
- **Counts (item 2):** one record-workload run per app (window H2) of the best build without the inline hit test
  (Redis's flags otherwise unchanged), on measure/hotspots `5bd1070eef6f`. The checked set and the hit condition are
  the shipped build's; every check reaches the runtime, which counts per caller pc the executed checks and the
  fast-path misses, and the classes of the missed addresses (stack, heap, TLS, other); -tsan-de-verified's inline
  tests count their executions through a call in front (`-tsan-site-count-calls`). MySQL: a 60 s run minus a 1 s
  run. FFmpeg: no run (work stopped); the 26 Sep profile of the same input, executed checks only.
- **Why (item 2):** the missed census's `-tsan-explain-missed` ported to 1738 (window H1), run with each app's best
  flags over its pre-TSan IR: per load and store, its status (kept, kept as DE's verified hit test, elided by which
  analysis) and, for EA, LO, STC, SWMR and DE, the verdict or the first reason it refused.
- **REAL or PHANTOM:** the 26 Sep O1/O2 oracles (the missed census's classes): T1 = touched by one thread only, T2 =
  consistently locked or written before publication, S = shared. T1 and T2 are PHANTOM (a static imprecision: the
  missing fact is the refusal the census names); S is REAL. Marked per row, since those profiles predate the record
  workloads.
- **Limits:** see the end.

## 1. Where the time goes

The perf split (3 Oct): `perf record -e cycles:u --call-graph=lbr`, 3 runs per app on the camera-ready trees
(root 1738), record workloads; MySQL's timed phase is each 30 s run minus its 1 s run. Shares are of the instrumented
run's user cycles (kernel time, e.g. memcached's sendmsg, is not in them).

Native time is not in these runs, and no leg has timed a camera-ready best configuration against native yet (a native arm is
planned for the camera-ready legs). Stock TSan over native, one leg each, N=1 per cell:

| app | stock / native | leg (view, root) |
|---|---|---|
| Redis | 5.99× (7 record commands; per command 4.3-9.8×) | Intel, natrd (view-host-redis-natrd), a081 |
| memcached V4 | 6.73×; the integrated arm in the same leg 1.223 over stock, so integrated / native ≈ 5.5× | AMD, mjv4, 317d |
| SQLite | 9.2× composite (stress2 4.5×, create_drop_index_1 16-21×) | AMD, natsq (view-apollo-sqlite-natsq), 9cd3 |
| MySQL Release | 7.5× (delete 11.2×, insert 7.9×, update 4.8×) | AMD, natmy (view-apollo-mysql-natmy), b86b |
| FFmpeg | 2.81× composite (copy 5.5-5.9×, mjpeg 6.0-6.2×, h264 1.3×, h265 1.4×) | Intel, natff, 78fe |

At these ratios the instrumentation's share of a run's time is 1 - 1/R: about 64-89 % (FFmpeg's h264/h265 about
20-30 %). So a category's share of the overhead is its share below, divided by that fraction (approximate, two legs).

Categories:
- checks: N1's inline hit tests and DE-verified sites, a runtime entry up to its first return, and misses (the rest of
  the entry, the miss entries, the race check);
- FE: function entry and exit, inline or called;
- ranges and interceptors;
- atomics and sync;
- runtime internals: slot preemption, global resets, trace part switches;
- the deadlock detector;
- program: uninstrumented work in the app's own code;
- other: libc, ld.so and uninstrumented libraries (FFmpeg's libx264/libx265).

| app (threads) | checks (inline hit / entry hit / miss) | FE | ranges + interceptors | atomics + sync | runtime internals | deadlock detector | program | other |
|---|---|---|---|---|---|---|---|---|
| Redis, main thread (12 % of cycles) | 27.8 (22.3 / 1.2 / 4.3) | 10.3 | **36.4** | 1.7 | 0.2 | 0.0 | 21.4 | 2.2 |
| Redis, I/O threads (88 %) | 0.0 | 0.0 | 0.1 | **98.5** (spin) | 0.0 | 0.0 | 1.4 | 0.0 |
| memcached (workers) | 29.5 (0.2 / 15.4 / 13.9) | 1.2 | **31.3** | **24.8** | 0.1 | 2.0 | 7.5 | 3.6 |
| SQLite | **50.6** (0.2 / 31.5 / 18.9) | 3.3 | 17.4 | 7.3 | 0.7 | 0.4 | 16.5 | 3.8 |
| MySQL, timed phase | **40.6** (0.2 / 22.6 / 17.8) | 12.1 | 7.3 | 14.8 | 0.5 | 0.1 | 23.0 | 1.6 |
| FFmpeg h264 | 17.1 (9.6 / 0.1 / 7.4) | 1.2 | 2.4 | 8.4 | 0.1 | 0.4 | 11.6 | 58.9 (x264) |
| FFmpeg h265 | 9.4 (5.1 / 0.1 / 4.2) | 0.7 | 21.1 | 1.1 | 0.0 | 0.1 | 6.3 | 61.3 (x265) |
| FFmpeg mjpeg | **58.4** (27.2 / 1.2 / 30.0) | 2.3 | 4.8 | 1.3 | 0.4 | 0.0 | 32.2 | 0.5 |
| FFmpeg copy | 3.5 (3.1 / 0.4 / 0.0) | 3.9 | **26.8** | 2.0 | 0.5 | 0.0 | 55.2 | 8.1 |

Readings:
- **Plain checks lead on SQLite, MySQL and FFmpeg's mjpeg**, with misses a third to a half of their check cycles.
  On memcached checks come second to ranges and interceptors.
- **Ranges and interceptors are the largest category no analysis touches.** Redis's main thread spends more there
  (36.4 %) than on all its plain checks (27.8 %); memcached 31.3 %, FFmpeg copy 26.8 %, h265 21.1 %, SQLite 17.4 %.
  Section 3.
- **memcached's atomics and sync, 24.8 %:** section 4.
- Runtime internals and the deadlock detector are small everywhere (≤ 2 %).
- Redis's I/O threads spin on atomics, off the main thread's path. They are not the main thread's cost; the phase
  guard (quiet mode) targets the main thread.

## 2. Per app: the plain checks that cost most

Each table lists the 15 locations with the most check cycles, from the perf split (inline hits, entry hits
and misses at that source location). Next to the cycles are the H2 counting run's executed checks and fast-path
misses at that location, as shares of the app's plain checks, with the location's own hit rate.

The other columns:
- **object:** where the missed addresses lay (stack, heap, TLS, other), or the 26 Sep flags where no miss was seen,
  and the global's name;
- **class:** the 26 Sep oracle's class (T1 one thread, T2 consistently locked or published, S shared), marked
  PHANTOM or REAL;
- **why:** the explain census's status, and the refusal that matters for the class: EA for T1, LO and SWMR for T2,
  EA for S, with STC and DE always;
- **unattributed:** a location at line 0 has no debug line, so the census cannot place it.

Atomic accesses are left out. TSan's atomics run through the same runtime path, so the counting runtime saw them,
but they are not plain checks. They are the Redis I/O threads' spin on `io_threads_pending` (55 % of Redis's counted
accesses) and memcached's mutex objects (4-6 %); they appear in section 4.

### Hit rates and class split (shares of plain checks)

| app | plain checks counted | hit rate | T1 (one thread) | T2 (locked / published) | S (shared) | no 26 Sep class |
|---|---|---|---|---|---|---|
| memcached | 27.3 G | **12.2 %** | 17.5 % | 10.0 % | 59.1 % | 13.3 % |
| Redis (main thread) | 8.1 G | 96.9 % | 47.5 % | 11.7 % | 33.7 % | 7.1 % |
| SQLite | 7.8 G | 71.3 % | 53.2 % | 20.8 % | 16.7 % | 9.3 % |
| MySQL (timed phase) | 40.9 G | 54.6 % | 22.0 % | 9.5 % | 62.5 % | 5.9 % |
| FFmpeg (26 Sep profile) | 52.0 G | n/a | 27.0 % | 42.6 % | 30.5 % | - |

- **memcached's checks are mostly misses.** Every request takes and releases mutexes (the item lock and the worker's
  stats lock), each a new epoch, so a check rarely finds its own record of the same epoch. Removing a check that hits
  saves its fast path (`tsan-fast-path-proof`); here most checks pay the slow path, so each removal is worth more.
- **Redis's main-thread checks almost always hit.** Their cost is the number of checks, not the misses.
- **MySQL** misses about half its checks, and 62.5 % of its checks touch genuinely shared memory (S): InnoDB's
  mutexes, latches and buffer pages.

### memcached (cmb; checks 29.5 % of cycles: entry 15.4, miss 13.9)

| # | cycles | location | function | r/w | executed | misses (hit rate) | object | class (26 Sep) | status | why still checked |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 5.39 % | `murmur3_hash.c:50` | `getblock32` | r | 26.62 % | 31.44 % (0.0 %) | heap global/other | S REAL | kept | EA: pointer-argument; STC: address-taken; DE: no-dominating-access |
| 2 | 1.49 % | `assoc.c:74` | `assoc_find` | r | 0.90 % | 1.06 % (0.0 %) | global/other global expanding | T2 PHANTOM | kept | LO: access-without-lock; SWMR: not-checked; STC: called-from-mt; DE: no-dominating-access |
| 3 | 1.41 % | `proto_text.c:582` | `process_get_command` | r | 0.82 % | 0.97 % (0.0 %) | global/other global settings | T2 PHANTOM | kept | LO: access-without-lock; SWMR: address-escapes; STC: called-from-mt; DE: different-location |
| 4 | 1.02 % | `items.c:1007` | `do_item_get` | r | 0.90 % | 1.06 % (0.0 %) | global/other global settings | T2 PHANTOM | kept | LO: access-without-lock; SWMR: address-escapes; STC: called-from-mt; DE: no-dominating-access |
| 5 | 0.99 % | `memcached.c:0` | `main` | r | 4.59 % | 3.30 % (39.2 %) | ? | - ? | ? | no debug line: unattributed |
| 6 | 0.79 % | `assoc.c:0` | `assoc_find` | ? | 0.90 % | 1.06 % (0.0 %) | ? | - ? | ? | no debug line: unattributed |
| 7 | 0.58 % | `proto_text.c:0` | `complete_nread_ascii` | r | 2.62 % | 2.13 % (31.2 %) | ? | - ? | ? | no debug line: unattributed |
| 8 | 0.57 % | `memcached.c:1002` | `conn_set_state` | r | 0.52 % | 0.32 % (47.2 %) | global/other global settings | T2 PHANTOM | kept | LO: access-without-lock; SWMR: address-escapes; STC: called-from-mt; DE: release-between |
| 9 | 0.42 % | `assoc.c:79` | `assoc_find` | r | 0.90 % | 1.07 % (0.0 %) | global/other global hashpower | T2 PHANTOM | kept | LO: access-without-lock; SWMR: not-checked; STC: called-from-mt; DE: different-location |
| 10 | 0.40 % | `assoc.c:87` | `assoc_find` | r | 0.40 % | 0.47 % (0.0 %) | global/other | S REAL | kept | EA: global-object; STC: called-from-mt; DE: different-location |
| 11 | 0.35 % | `memcached.c:2698` | `_transmit_post` | r | 0.90 % | 1.07 % (0.0 %) | heap | T1 PHANTOM | kept | EA: pointer-argument; STC: creates-threads; DE: different-location (*0) |
| 12 | 0.31 % | `assoc.c:79` | `assoc_find` | r | 0.90 % | 1.06 % (0.0 %) | global/other global primary_hashtable | T2 PHANTOM | kept | LO: access-without-lock; SWMR: not-checked; STC: called-from-mt; DE: different-location |
| 13 | 0.30 % | `memcached.c:1187` | `resp_start` | w | 0.90 % | 1.06 % (0.0 %) | heap | T1 PHANTOM | kept | EA: pointer-argument; STC: called-from-mt; DE: different-location (*0) |
| 14 | 0.30 % | `memcached.c:0` | `drive_machine` | ? | 0.84 % | 0.99 % (0.0 %) | ? | - ? | ? | no debug line: unattributed |
| 15 | 0.30 % | `memcached.c:1190` | `resp_start` | r | 0.90 % | 1.06 % (0.0 %) | heap | S REAL | kept | EA: pointer-argument; STC: called-from-mt; DE: different-location |

- **#1, `getblock32` in murmur3 (26.6 % of plain checks, 5.4 % of cycles, 0 % hits):** reads the request key in
  the connection's read buffer.
  - The 26 Sep oracle calls the bytes shared, because read buffers come from a per-thread cache and are reused. But
    EVCONF already confines the buffer to the worker's event loop (`conn.rbuf` and `conn.rcurr` in its spec).
  - The fact stops at the call `hash(key, nkey)`: EA sees a pointer argument, and EVCONF does not follow pointers
    into callees.
  - New candidate **EVCONF-ARGS**: carry EVCONF's confinement into the callee's pointer argument (here a
    leaf hash function). Ceiling about 5.4 % of cycles.
- **#2-#4 and similar:** the global flags `expanding`, `hashpower` and `settings.*`, read without a lock.
  - `settings` is written at start-up and by admin commands; `expanding` and `hashpower` by the hash maintainer
    under its lock.
  - SWMR refuses them (address escapes, or not checked); LO sees no lock at the read.
  - The T2 mass is 10 % of checks. SWMR-G (write-once globals) was closed by count on the old workloads; on this
    workload, memcached's `settings` reads alone are about 3 % of checks. This is queue item "memcached globals" of
    1 Oct; closed 4 Oct: `hashpower` and `expanding = true` are written under the pause of every thread,
    `expanding = false` under one item lock (a genuine race), and the `settings` fields that admin commands write
    have no static route (see the T2 row of section 5).
- **Function entry and the read buffer's other bytes** (`try_read_command_ascii`, `tokenize_command`): EVCONF covers
  the fields but not the interceptors' ranges over the buffer (section 3).

### Redis (crb; main thread; checks 27.8 % of its cycles, 22.3 inline hits)

| # | cycles | location | function | r/w | executed | misses (hit rate) | object | class (26 Sep) | status | why still checked |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 3.08 % | `src/networking.c:348` | `_addReplyProtoToList` | r | 2.53 % | 0.00 % (100.0 %) | heap | S REAL | kept | EA: pointer-argument; STC: no-main-in-unit; DE: no-dominating-access |
| 2 | 1.78 % | `src/quicklist.c:1300` | `quicklistNext` | w | 1.27 % | 0.00 % (100.0 %) | stack | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: no-dominating-access |
| 3 | 1.35 % | `src/networking.c:362` | `_addReplyProtoToList` | w | 1.27 % | 0.00 % (100.0 %) | heap | T2 PHANTOM | kept | LO: not-a-global; SWMR: not-a-global; STC: no-main-in-unit; DE: read-cannot-cover-write (*0) |
| 4 | 0.98 % | `src/t_list.c:124` | `listTypeNext` | r | 1.27 % | 0.00 % (100.0 %) | (26 Sep: heap) | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: no-dominating-access |
| 5 | 0.98 % | `src/quicklist.c:1341` | `quicklistNext` | w | 0.42 % | 0.00 % (100.0 %) | (26 Sep: stack) | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location |
| 6 | 0.89 % | `src/listpack.c:1164` | `lpBytes` | r | 0.45 % | 0.45 % (98.5 %) | heap | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location |
| 7 | 0.78 % | `src/networking.c:334` | `_addReplyToBuffer` | r | 2.58 % | 0.00 % (100.0 %) | (26 Sep: heap) | S REAL | kept | EA: pointer-argument; STC: no-main-in-unit; DE: no-dominating-access |
| 8 | 0.60 % | `src/networking.c:411` | `addReply` | r | 0.44 % | 0.00 % (100.0 %) | heap | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location |
| 9 | 0.55 % | `src/listpack.c:390` | `lpDecodeBacklen` | r | 0.44 % | 11.49 % (61.7 %) | heap | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: no-dominating-access |
| 10 | 0.55 % | `src/networking.c:384` | `_addReplyToBufferOrList` | r | 1.29 % | 0.00 % (100.0 %) | (26 Sep: heap) | S REAL | kept | EA: pointer-argument; STC: no-main-in-unit; DE: no-dominating-access |
| 11 | 0.53 % | `src/quicklist.c:1318` | `quicklistNext` | r | 0.42 % | 0.00 % (100.0 %) | (26 Sep: heap) | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location (*0) |
| 12 | 0.53 % | `src/quicklist.c:1330` | `quicklistNext` | r | 0.42 % | 0.00 % (100.0 %) | (26 Sep: heap) | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location |
| 13 | 0.49 % | `src/listpack.c:477` | `lpNext` | r | 0.43 % | 8.41 % (71.1 %) | heap | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location |
| 14 | 0.49 % | `src/networking.c:938` | `addReplyLongLongWithPrefix` | w | 0.01 % | 0.38 % (23.0 %) | stack | T1 PHANTOM | kept | EA: passed-to-call; STC: no-main-in-unit; DE: different-location |
| 15 | 0.46 % | `src/listpack.c:0` | `lpValidateNext` | ? | - | 0.00 % (n/a) | ? | - ? | ? | no debug line: unattributed |

- **Rows 1 and 3:** `_addReplyProtoToList` reads the reply list's tail block and writes its `used` count. The
  object is the client's reply list, which the main thread fills and the I/O threads drain (S and T2 by the oracle).
  Quiet mode (the Redis phase guard) is the lever: the main thread skips while the I/O threads are quiet.
- **Rows 2, 4, 5 and the rest of `quicklistNext`/`listTypeNext`:** the LRANGE iterator, a `quicklistEntry` on the
  caller's stack (T1). EA refuses it as a pointer argument. On the main thread these are in quiet mode's reach; as
  a static fact they are the own-stack class (OWN-STACK, parked: sound form ≈ −1 % on MySQL).
- Everything on Redis's main thread is in quiet mode's reach. Its plain-check cost (27.8 %) plus ranges (36.4 %) is
  what the mode can take, less what stays checked on registered objects.

### SQLite (cq7; checks 50.6 % of cycles: entry 31.5, miss 18.9; flat, top site 0.8 %)

| # | cycles | location | function | r/w | executed | misses (hit rate) | object | class (26 Sep) | status | why still checked |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 1.57 % | `build/sqlite3.c:0` | `sqlite3_str_vappendf` | w | 3.79 % | 3.21 % (75.5 %) | ? | - ? | ? | no debug line: unattributed |
| 2 | 0.77 % | `build/sqlite3.c:37434` | `strHash` | r | 0.68 % | 1.99 % (15.4 %) | stack heap global/other | T2 PHANTOM | kept | LO: not-a-global; SWMR: not-a-global; STC: no-main-in-unit; DE: different-location |
| 3 | 0.71 % | `build/sqlite3.c:35933` | `sqlite3StrICmp` | r | 0.62 % | 1.60 % (25.2 %) | stack heap global/other | T2 PHANTOM | kept | LO: not-a-global; SWMR: not-a-global; STC: no-main-in-unit; DE: no-dominating-access |
| 4 | 0.52 % | `build/sqlite3.c:95322` | `sqlite3VdbeExec` | r | 1.74 % | 1.57 % (74.0 %) | heap | S REAL | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location (*0) |
| 5 | 0.45 % | `build/sqlite3.c:35934` | `sqlite3StrICmp` | r | 0.56 % | 1.04 % (45.9 %) | heap global/other | T2 PHANTOM | kept | LO: not-a-global; SWMR: not-a-global; STC: no-main-in-unit; DE: different-location |
| 6 | 0.43 % | `build/sqlite3.c:178052` | `yy_shift` | w | 0.33 % | 0.95 % (17.7 %) | stack | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location |
| 7 | 0.43 % | `build/sqlite3.c:29583` | `sqlite3_mutex_leave` | r | 0.20 % | 0.71 % (0.0 %) | global/other global sqlite3Config | T2 PHANTOM | kept | LO: address-in-initializer; SWMR: address-in-initializer; STC: no-main-in-unit; DE: different-location |
| 8 | 0.40 % | `build/sqlite3.c:90960` | `vdbeRecordCompareString` | r | 0.83 % | 0.39 % (86.2 %) | heap | T2 PHANTOM | kept | LO: not-a-global; SWMR: not-a-global; STC: no-main-in-unit; DE: no-dominating-access |
| 9 | 0.40 % | `build/sqlite3.c:31881` | `sqlite3_str_vappendf` | r | 0.44 % | 1.50 % (1.3 %) | global/other | T2 PHANTOM | kept | LO: not-a-global; SWMR: not-a-global; STC: no-main-in-unit; DE: different-location |
| 10 | 0.37 % | `build/sqlite3.c:181850` | `sqlite3GetToken` | r | 0.36 % | 1.22 % (1.0 %) | heap global/other | T2 PHANTOM | kept | LO: not-a-global; SWMR: not-a-global; STC: no-main-in-unit; DE: different-location |
| 11 | 0.35 % | `build/sqlite3.c:97352` | `sqlite3VdbeExec` | r | 0.20 % | 0.63 % (10.6 %) | heap | T2 PHANTOM | kept | LO: not-a-global; SWMR: not-a-global; STC: no-main-in-unit; DE: different-location |
| 12 | 0.32 % | `build/sqlite3.c:97430` | `sqlite3VdbeExec` | r | 0.53 % | 1.85 % (0.0 %) | heap | T2 PHANTOM | kept | LO: not-a-global; SWMR: not-a-global; STC: no-main-in-unit; DE: different-location (*0) |
| 13 | 0.28 % | `build/sqlite3.c:90975` | `vdbeRecordCompareString` | r | 0.83 % | 0.39 % (86.4 %) | heap | T2 PHANTOM | kept | LO: not-a-global; SWMR: not-a-global; STC: no-main-in-unit; DE: different-location |
| 14 | 0.27 % | `build/sqlite3.c:29557` | `sqlite3_mutex_enter` | r | 0.23 % | 0.78 % (1.5 %) | global/other global sqlite3Config | T2 PHANTOM | kept | LO: address-in-initializer; SWMR: address-in-initializer; STC: no-main-in-unit; DE: different-location |
| 15 | 0.26 % | `build/sqlite3.c:36921` | `sqlite3GetVarint32` | r | 0.53 % | 0.48 % (73.8 %) | heap | T2 PHANTOM | kept | LO: not-a-global; SWMR: not-a-global; STC: no-main-in-unit; DE: different-location (*0) |

- **Flat.** No location reaches 1 % of cycles except one without a debug line (`sqlite3_str_vappendf`). The mass is
  in the classes: T1 53 % (EA pointer argument 45 %, object from call 7 %) and T2 21 %, all on non-globals (LO and
  SWMR judge globals only).
- **What the T1 and T2 objects are:** the parser's stack `Parse` and engine (`strHash` and `sqlite3StrICmp` hash and
  compare identifiers from the SQL text and the schema), the VDBE's registers, and b-tree page content.
  - Those under BtShared's mutex are LO-OBJ-G's (spec v7).
  - Those under the connection's `db->mutex` (the VDBE under construction, the sorter, lookaside) have no spec
    entry: an LO-OBJ-G extension to `db->mutex` (section 6).

### MySQL (cyb, timed phase; checks 40.6 % of cycles: entry 22.6, miss 17.8; flat, top site 0.36 %)

| # | cycles | location | function | r/w | executed | misses (hit rate) | object | class (26 Sep) | status | why still checked |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 0.36 % | `include/ut0mutex.h:114` | `void mutex_enter_inline<PolicyMu` | r | 0.22 % | 0.48 % (1.1 %) | global/other global srv_n_spin_wait_rounds | T2 PHANTOM | kept | LO: named-outside-unit; SWMR: named-outside-unit; STC: no-main-in-unit; DE: no-dominating-access |
| 2 | 0.26 % | `std_thread.h:97` | `std::thread::id::id()` | w | 0.56 % | 0.98 % (18.9 %) | stack global/other | S REAL | kept | EA: pointer-argument; STC: no-main-in-unit; DE: no-dominating-access |
| 3 | 0.20 % | `include/ib0mutex.h:732` | `PolicyMutex<TTASEventMutex<Gener` | r | 0.23 % | 0.49 % (0.0 %) | stack heap global/other | S REAL | kept | EA: pointer-argument; STC: no-main-in-unit; DE: no-dominating-access |
| 4 | 0.20 % | `std_thread.h:100` | `std::thread::id::id(unsigned lon` | w | 0.39 % | 0.81 % (2.3 %) | stack | S REAL | kept | EA: pointer-argument; STC: no-main-in-unit; DE: no-dominating-access |
| 5 | 0.19 % | `include/mach0data.ic:73` | `mach_read_from_2(unsigned char c` | r | 0.39 % | 0.48 % (43.1 %) | heap global/other | S REAL | kept | EA: pointer-argument; STC: no-main-in-unit; DE: no-dominating-access |
| 6 | 0.19 % | `ut0ut.cc:105` | `ut_delay(unsigned long)` | r | 0.19 % | 0.10 % (74.7 %) | global/other global _ZN2ut26spin_wait_pause_mult | T2 PHANTOM | kept | LO: no-lock-functions-in-unit; SWMR: named-outside-unit; STC: no-main-in-unit; DE: no-dominating-access |
| 7 | 0.17 % | `unique_ptr.h:199` | `std::__uniq_ptr_impl<char, void ` | r | 0.47 % | 0.51 % (49.8 %) | stack heap global/other | S REAL | kept | EA: object-from-call; STC: no-main-in-unit; DE: no-dominating-access |
| 8 | 0.16 % | `include/ib0mutex.h:733` | `PolicyMutex<OSTrackMutex<Generic` | r | 0.22 % | 0.46 % (0.0 %) | heap | S REAL | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location (*0) |
| 9 | 0.15 % | `include/mach0data.ic:65` | `mach_read_from_1(unsigned char c` | r | 0.22 % | 0.29 % (38.3 %) | stack heap global/other | S REAL | kept | EA: pointer-argument; STC: no-main-in-unit; DE: no-dominating-access |
| 10 | 0.15 % | `ha_innodb.cc:2021` | `thd_to_innodb_session(THD*)` | r | 0.08 % | 0.15 % (11.5 %) | global/other global _ZL15innodb_hton_ptr | T2 PHANTOM | kept | LO: access-without-lock; SWMR: written-in-mt; STC: no-main-in-unit; DE: no-dominating-access |
| 11 | 0.14 % | `include/dict0mem.h:1348` | `dict_index_t::has_row_versions()` | r | 0.39 % | 0.07 % (91.5 %) | heap | S REAL | kept | EA: pointer-argument; STC: no-main-in-unit; DE: no-dominating-access |
| 12 | 0.14 % | `include/trx0trx.h:1470` | `TrxInInnoDB::enter(trx_t*, bool)` | r | 0.07 % | 0.14 % (11.1 %) | global/other global srv_read_only_mode | T2 PHANTOM | kept | LO: named-outside-unit; SWMR: named-outside-unit; STC: no-main-in-unit; DE: no-dominating-access |
| 13 | 0.14 % | `include/dict0dict.ic:134` | `dict_index_is_spatial(dict_index` | r | 0.21 % | 0.06 % (86.1 %) | heap | S REAL | kept | EA: pointer-argument; STC: no-main-in-unit; DE: no-dominating-access |
| 14 | 0.14 % | `include/dict0mem.h:1314` | `dict_index_t::is_clustered() con` | r | 0.23 % | 0.07 % (86.4 %) | heap | S REAL | kept | EA: pointer-argument; STC: no-main-in-unit; DE: no-dominating-access |
| 15 | 0.13 % | `pfs.cc:5875` | `pfs_start_stage_v1(unsigned int,` | r | 0.07 % | 0.12 % (14.2 %) | global/other global flag_global_instrumentation | S REAL | kept | EA: global-object; STC: no-main-in-unit; DE: different-location |

- **Flat**, and 62.5 % of plain checks are shared (S): InnoDB's mutex implementation (`ut0mutex.h`, `ib0mutex.h`),
  thread ids built on the stack (`std::thread::id`) and buffer-pool page reads (`mach_read_from_2`).
- The closed MySQL levers (WP summaries, top-down facts, ODR trust, InnoDB latch census, all ≤ 0.14 % or 3.2 %) stand.
  No new class appears.

### FFmpeg (cfb; 26 Sep profile of the same input, no counting run; checks 9-58 % of cycles by codec)

| # | cycles | location | function | r/w | executed | misses (hit rate) | object | class (26 Sep) | status | why still checked |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 0.94 % | `h264_cabac.c:1759` | `decode_cabac_residual_dc_interna` | w | 2.19 % | n/a | global decode_cabac_residual_internal.coeff_abs_ | T2 PHANTOM | kept | LO: not-a-global; SWMR: not-a-global; STC: no-main-in-unit; DE: different-location |
| 2 | 0.60 % | `mjpegenc.c:143` | `record_block` | w | 0.97 % | n/a | (26 Sep: heap/other) | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location (*0) |
| 3 | 0.53 % | `mjpegenc_common.c:494` | `ff_mjpeg_encode_stuffing` | r | 0.97 % | n/a | (26 Sep: heap/other) | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location (*0) |
| 4 | 0.46 % | `mjpegenc_common.c:385` | `ff_mjpeg_encode_picture_frame` | r | 0.97 % | n/a | (26 Sep: heap/other) | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location (*0) |
| 5 | 0.44 % | `h264_slice.c:2479` | `loop_filter` | r | 0.07 % | n/a | (26 Sep: heap) | S REAL | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location (*0) |
| 6 | 0.42 % | `h264_slice.c:2283` | `loop_filter` | w | 0.03 % | n/a | (26 Sep: heap) | T2 PHANTOM | kept | LO: not-a-global; SWMR: not-a-global; STC: no-main-in-unit; DE: different-location |
| 7 | 0.42 % | `mjpegenc.c:142` | `record_block` | w | 0.97 % | n/a | (26 Sep: heap/other) | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location (*0) |
| 8 | 0.42 % | `mjpegenc.c:141` | `record_block` | w | 0.97 % | n/a | (26 Sep: heap) | S REAL | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location (*0) |
| 9 | 0.41 % | `mjpegenc.c:170` | `record_block` | w | 0.80 % | n/a | (26 Sep: heap/other) | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location (*0) |
| 10 | 0.40 % | `mjpegenc_common.c:493` | `ff_mjpeg_encode_stuffing` | r | 0.97 % | n/a | (26 Sep: heap/other) | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location (*0) |
| 11 | 0.34 % | `swscale.c:182` | `lumRangeToJpeg_c` | r | 1.51 % | n/a | (26 Sep: heap) | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: no-dominating-access |
| 12 | 0.32 % | `swscale.c:182` | `lumRangeToJpeg_c` | w | 1.53 % | n/a | (26 Sep: heap) | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location |
| 13 | 0.28 % | `mjpegenc_common.c:402` | `ff_mjpeg_encode_picture_frame` | r | 0.80 % | n/a | (26 Sep: heap/other) | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location (*0) |
| 14 | 0.27 % | `mjpegenc_common.c:397` | `ff_mjpeg_encode_picture_frame` | r | 0.97 % | n/a | (26 Sep: heap/other) | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location (*0) |
| 15 | 0.25 % | `mjpegenc_common.c:396` | `ff_mjpeg_encode_picture_frame` | r | 0.97 % | n/a | (26 Sep: heap/other) | T1 PHANTOM | kept | EA: pointer-argument; STC: no-main-in-unit; DE: different-location (*0) |

- The cost is in mjpeg (checks 58.4 % of its cycles) and h264 decoding (CABAC): codec contexts reached through
  pointer arguments (T1, EA) and per-slice state (T2, not a global).
- h264 and h265 are dominated by uninstrumented libx264 and libx265 (about 60 % of cycles).
- FFmpeg work is stopped; this is for the record only.

## 3. Ranges and interceptors

Plain checks are flat: the hottest plain-check site is memcached's murmur3 key read at 5.4 % of cycles, and SQLite's
and MySQL's are under 1.6 %. Ranges are not flat: two call sites carry most of their app's range cost. Source:
the entry split (4 Oct). Each runtime entry's
cycles are attributed to the first instrumented frame (its LBR caller). Shares are of the app's cycles (Redis: of the
main thread's).

### The two sites that dominate

| site | share | object and owner | why nothing covers it today | the fact that would |
|---|---|---|---|---|
| memcached `resp_allocate`: `memset(resp, 0, sizeof(*resp))`, memcached.c:1085 (1118 for a new bundle) | **16.6 %** of all cycles | an `_mc_resp` (1,176 bytes: `wbuf[1024]` plus headers) in the worker's own bundle (`th->open_bundle`); owned by the worker's event loop | EVCONF (memcached's best) treats response objects as event-loop confined, but its guard covers only plain loads and stores of the listed fields; `instrumentMemIntrinsic` never asks it. EA cannot see the ownership (a heap object reached through the thread struct) | **EVCONF-RANGES** (in work, 4 Oct): EVCONF's confinement applied to memory intrinsics on a confined object, destination and source. An UNSOUND ceiling oracle (cmb with the two memsets uninstrumented) is queued in the timing plan; the confinement argument for every byte of `_mc_resp` goes to review |
| Redis `_addReplyProtoToList`: `memcpy(tail->buf + tail->used, s, copy)`, networking.c:361 | **14.1 %** of the main thread | the tail `clientReplyBlock` of `c->reply`, a heap object of the client: the main thread fills it while executing commands; the I/O threads read it in `writev` during their write phase | genuinely handed between threads, by phase, so no static fact holds | **quiet mode's range skip** (`phase_guard_ranges`, eee8e8455ebe, tests 6da72199d92e): while the I/O threads are quiet, the main thread skips its range checks except on registered objects |

### By app: the entries above 1 %, their callers, and their class

Classes:
- *thread-local*: the object is private, but no analysis proves it;
- *covered*: a fact we already have covers the object, but that fact is applied only to plain accesses;
- *shared*: genuinely shared;
- *allocator*: TSan's allocation bookkeeping, not a check;
- *outside*: the call comes from uninstrumented code.

| app | entry (share) | top callers | class |
|---|---|---|---|
| Redis main | memcpy 15.9 | networking.c:361 (14.1, above), object.c:105 createEmbeddedStringObject (1.2: a fresh object) | phase-shared → quiet mode; fresh object → thread-local (EA: object-from-call) |
| | memset 5.0 | quicklist.c:1300 `initEntry` (4.8): 16 bytes of the caller's stack `quicklistEntry` | thread-local (EA refuses a pointer argument); quiet mode's range skip covers it on the main thread |
| | malloc/free/realloc 8.2 (+ malloc_usable_size 0.9) | zmalloc | allocator |
| | memmove 1.9 | lpInsert (listpack, keyspace data only the main thread touches) | thread-local in practice (T1 role) → quiet mode |
| | writev 0.9, strchr 0.9 | connSocketWritev (reply blocks), processMultibulkBuffer (the query buffer the I/O threads fill) | phase-shared → quiet mode |
| memcached | memset 16.9 | resp_allocate (16.6, above) | covered (EVCONF) → EVCONF-RANGES |
| | memchr 2.1, strlen 1.6, read 1.8 | try_read_command_ascii, tokenize_command, tcp_read: all on the connection's read buffer `rbuf` | covered: EVCONF confines `conn.rbuf`/`rcurr`, but not the interceptors' ranges over the buffer → EVCONF-RANGES (interceptors) |
| | bcmp 2.0 | assoc_find: the request key (rbuf) against the item's key (hash table) | half covered (key), half shared (item) |
| | memcpy 1.2 | do_item_alloc (0.6: a fresh item before it is linked), _transmit_pre (0.4) | fresh item: thread-local until linked (no fact); _transmit_pre: unclassified |
| SQLite | memcpy 4.3 | sqlite3VdbeExec (2.0: VDBE register copies), sqlite3_str_append (0.4) | VDBE registers belong to the statement's connection, under its mutex: a lock fact (LO-OBJ-G's kind) not applied to ranges |
| | memcmp 4.0 | vdbeRecordCompareString (3.7): record bytes on a b-tree page against the caller's key | covered: LO-OBJ-G puts page content under the BtShared mutex for plain accesses, not for memcmp's ranges |
| | malloc/free 3.6, memset 2.2 (scattered), pwrite 0.7 | sqlite3MemMalloc/Free, seekAndWriteFd | allocator; I/O |
| MySQL timed | memset 2.6, memcpy 2.2 | flat: MDL_key constructor (0.6, a stack object), dyn_buf_t, PFS statement copies, empty_record | mostly thread-local constructions (EA refuses `this` as a pointer argument) |
| FFmpeg h264 / h265 | memcpy 14.2, memset 5.6 (h265); mutexes 7.5 (h264) | libx265's CUData copies, libx264's thread pool | outside: uninstrumented libraries' libc calls, intercepted. TSan's `ignore_noninstrumented_modules` would drop them, but a race between such a call and instrumented code is one stock reports |
| FFmpeg mjpeg | memcpy 3.4 | load_input_picture (2.8): the input frame into the encoder's picture | shared (frame buffers passed between threads) |
| FFmpeg copy | free 7.7, posix_memalign 5.9, realloc 2.1, strcmp 5.8 | av_free/av_malloc; av_opt_find2 and `_init` | allocator; option parsing at start-up |

## 4. Atomics and synchronisation

- **memcached, 24.8 % of cycles.** Mutex interceptors take 24.7: `pthread_mutex_lock` 17.1 and `unlock` 7.6, plus
  1.6 of their own frames counted under ranges. The deadlock detector is a separate 2.0 %, so about 8 % of the
  mutex-related cost; the rest is the clock work of acquire and release and the interceptor.
  - `item_lock`/`item_unlock` (thread.c:122/138): 7.5. The global, hash-partitioned item lock array, taken by
    every worker: shared, real synchronisation.
  - `THR_STATS_LOCK` in resp_start/resp_free and process_get_command (memcached.c:1171/1188, proto_text.c:658):
    about 6.4. The worker's own `stats.mutex`, taken by its owner on every response and only rarely by the stats
    aggregator: synchronisation on a mostly owned object.
  - Nothing in the pass can remove a mutex operation. A cheaper owner-to-owner path, where the last releaser
    reacquires, is a runtime-track item (runtime work parked since 25 Sep).
- **SQLite, 7.3 %.** Its own mutexes (pthreadMutexEnter/Leave 6.2: shared cache, pcache, memory): real
  synchronisation.
- **MySQL, 14.8 %.** `std::atomic` loads 6.5 (thread ids, InnoDB counters and flags), compare-exchange 1.5, stores
  1.3, fetch_xor 0.8; mutexes small. Real atomics, each a runtime call. N1-ATOMIC (inline relaxed atomics) was
  closed by timing.
- **Redis, I/O threads.** 98.5 % of their cycles are an atomic spin (`__tsan_atomic64_load` while waiting for
  work). It is off the main thread's path and appears in "all threads" only.
- **FFmpeg h264.** Mutexes 7.5, almost all from libx264's own threads (outside).

## 5. Blocker classes across apps

Mass is given as shares of plain checks (section 2's split) where the class is about checks, and as shares of cycles
where it is about ranges, synchronisation or function entry. Each class is marked closed (with what closed it),
parked (with the decision), in work, or NEW (with a ceiling and what it would take).

| class | memcached | Redis main | SQLite | MySQL | FFmpeg | standing |
|---|---|---|---|---|---|---|
| EA refuses an object reached through a pointer argument (T1) | 16.9 % (+11.9 % unclassified) | 37.2 % | 45.2 % | 17.0 % | 20.9 % | static routes **closed** (top-down facts ≤ 0.14 % on all five apps, 2 Oct; WP summaries; CLONE-ESC). Run-time routes: quiet mode (Redis main, **in work**); own stack (OWN-STACK, **parked** 2 Oct: sound form ≈ −1 %). **NEW: EVCONF-ARGS** for memcached's hashed key (≤ 5.4 % of cycles) |
| EA: object returned by a call (T1) | 0.3 % | 2.2 % | 6.9 % | 2.5 % | 0 | **closed** (no allocator wrapper carries malloc/noalias; named allocators gave nothing) |
| LO and SWMR judge globals only (T2, not a global) | 0.9 % | 10.8 % | 17.6 % | 6.7 % | 42.5 % | LO-OBJ-G in SQLite's best (BtShared, spec v7); its extensions SQLITE-KEY **parked** (2 Oct); PERELEM **parked**. **NEW: LO-OBJ-G for `db->mutex` objects** (section 6) |
| globals read without a lock, or whose address escapes (T2 globals) | 7.7 % (`settings`, `expanding`, `hashpower`) | 0.2 % | 1.7 % (address in an initializer) | 2.7 % (named outside the unit) | 0 | SWMR-G and per-field SWMR **closed** by count on the old workloads (< 3 %); for memcached **closed** 4 Oct (the oracle o2 adds 4.4 % on Intel): the admin-written `settings` fields (3.07 % of checks) and `expanding` (1.06 %, a genuine benign race at an expansion's end) have no sound route and stay checked; `hashpower` (1.07 %) is ordered by the pause protocol, which no assertion states (not taken); the never-written `settings` fields (0.34 %) are below the bar (libevent-confinement.md) |
| genuinely shared, DE has no cover at the same location (S) | 52.8 % | 31.0 % | 15.7 % | 57.8 % | 21.9 % | REAL. The old same-location rule is UNSOUND (it lost a real memcached race, 3 Oct). Sound subsets DE-2R (merge adjacent) and DE-3R (loop ranges) were timed on the old workloads only; DE-AV (equal addresses) has its time ceiling ordered. On memcached 88 % of these checks miss, so each removal saves a slow path |
| ranges and interceptors (cycles) | 31.3 % | 36.4 % | 17.4 % | 7.3 % | copy 26.8 %, h265 21.1 % | **NEW: EVCONF-RANGES** (memcached ≤ 16.6 % plus the read-buffer interceptors ≈ 5.5 %; oracle queued, argument and pass change in work); quiet mode's range skip (Redis main ≈ 19 %; eee8e8455ebe, **in work**); **NEW: LO-OBJ-RANGES** (SQLite's memcmp and VDBE memcpy on lock-owned bytes, ≤ 5.7 %); allocator and uninstrumented callers: runtime or none |
| mutexes and atomics (cycles) | 24.8 % (deadlock detector 2.0) | 1.7 % | 7.3 % | 14.8 % | h264 8.4 % (libx264) | real synchronisation. An owner-to-owner mutex path (memcached's per-thread stats mutex, ≈ 6 %) is a runtime-track item, **parked** with the runtime track (25 Sep); N1-ATOMIC **closed** by timing |
| function entry and exit (cycles) | 1.2 % | 10.3 % | 3.3 % | 12.1 % | ≤ 2.3 % | FE-INL in the best; FE-HOT, FE-PM, FE-LAZY **closed** by timing; FE-NOFRAME **not taken** (changes report stacks) |

### New candidates, by ceiling

| candidate | app | ceiling | what it takes | status |
|---|---|---|---|---|
| EVCONF-RANGES | memcached | ≤ 16.6 % of cycles (resp_allocate's memset), plus ≈ 5.5 % of read-buffer interceptors | the confinement argument for every byte of `_mc_resp` (for review); `evconfCovered` applied in `instrumentMemIntrinsic` (destination and source); gate; audit | **measured +13.9 % Intel, +10.3 % AMD (4 Oct)**: cmg (sound, `-tsan-evconf-ranges`, constant lengths inside a typed object; audits A46, A46b, A46c) 1.139 over cmb, against the UNSOUND oracle cmo 1.141; the same root without the flag 0.987, A/A 0.988-1.001 (libevent-confinement.md) |
| quiet mode's range skip | Redis | ≈ 19 % of the main thread (networking.c:361 14.1, quicklist.c:1300 4.8) | already written (`phase_guard_ranges`, eee8e8455ebe; tests 6da72199d92e) | in work with the phase guard |
| LO-OBJ-RANGES | SQLite | ≤ 5.7 % of cycles (memcmp 3.7, VDBE memcpy 2.0) | LO-OBJ-G's guard applied to memory intrinsics and to the memcmp interceptor on spec-covered objects | **OUT** (4 Oct): reaches neither site. The memcpy copies VDBE registers, which `sqlite3_value_dup` reads without `db->mutex`, so they cannot be a root. The memcmp's page side reaches `vdbeRecordCompareString` as `const void *` through a function pointer, with no owner path (lo-obj-g-design.md §6.17) |
| EVCONF-ARGS | memcached | ≤ 5.4 % of cycles (26.6 % of checks) | EVCONF's confinement carried into a callee's pointer argument (`hash(key, nkey)`) | **oracle 1.082 over cmf** (Intel, 4 Oct; A/A 0.979-0.999); design for audit in libevent-confinement.md, audit A50 running |
| memcached's `settings`, `expanding`, `hashpower` reads | memcached | +4.4 % (oracle o2 over o1, Intel) | - | **closed** (4 Oct): no sound route except the never-written `settings` fields (0.34 % of checks, below the bar); see the T2 row |
| LO-OBJ-G on `db->mutex` | SQLite | ≈ 6-7 % of create_drop_index_1's checks (section 6) | spec entries for the connection's objects (VDBE, sorter, lookaside) under `db->mutex` | NEW |
| EVCONF-INTERCEPT | memcached | ≈ 4.7 % of cycles (memchr 2.1, strlen 1.6, bcmp's request-key half of 2.0); read 1.8 only with an entry that keeps the fd acquire | EVCONF's guard around the libc calls on `rbuf`/`rcurr` (EVCONF-RANGES' interceptor toggle); IN-BOUNDS for the variable lengths (strlen stops at the NUL the parser writes over the `\n` that memchr found within `rbytes`); bcmp checks the item's key explicitly | **oracle 1.065 over cmx** (Intel V4, 4 offsets, 1.039-1.083; A/A 1.007; leg ormt, 5 Oct): memchr, strlen and bcmp's request-key side, read() left out. X5 (sendmsg iovecs, `intercept_send=0` on top) adds 1.007, inside the A/A. Next: the pass change and its delta audit |
| EVCONF-FIELDS (X1) | memcached | struct conn reads ≥ 14.0 % of plain checks, 77 % misses (`c->thread` 4.05); writes 3.6 % stay checked | a new claim kind, owner-reads: the owner's reads of a conn field are unchecked while G holds, its writes stay checked; a per-field list of writers outside the owner's loop | **oracle 1.036 over cmx** (Intel V4, 4 offsets, 1.026-1.048; A/A 1.007; leg ormt, 5 Oct; tree cmk__: every conn read dropped). Next: the code-fact list and the claim kind, for audit |
| LO-OBJ-ARGS | SQLite | ≤ 4.4 % of cycles (memcmp 3.7 under `vdbeRecordCompareString`, its record-side plain reads 0.68) | EVCONF-ARGS' clone and run-time dispatch at the `xRecordCompare` sites whose record is a page cell, with LO-OBJ-G's lock-held guard in the clone; the overflow copy keeps the original; no new premise | NEW (4 Oct): EVCONF-ARGS' dispatch removes LO-OBJ-RANGES' blocker (the function pointer). UNSOUND ceiling tree cqa__ (cq7 + skip and toggle lists, every caller, so it overstates) in its build window; then cq7 \| cqa \| A/A, apollo B |
| Redis I/O threads' client buffers | Redis | ≈ 0: the I/O threads' cycles are 98.5 % the spin on `io_threads_pending`, not checks | - | **closed** (4 Oct): nothing to take on the I/O threads; the main thread's side of the same buffers is quiet mode's (row above) |
| T1 with a run-time owner tag | all | T1 is 17-53 % of plain checks; its unsound ceiling is measured (the 26 Sep O1-all arms) | - | **closed** (4 Oct), unsound by construction: skipping the owner's access drops its shadow record, so a later foreign access races with nothing. Restoring the records at publication needs each skipped access's epoch, and keeping it is the shadow update the skip saves (the fast-path proof's corollary) |
| MySQL THD owner-reads (X3, THD-READS) | MySQL | THD-field reads ≤ 2.35 % of plain checks (through a THD 1.74, through its sub-objects 0.61; writes 0.57), by source text at each H2 site; the named session-arena classes (Item, parser, LEX, plan, Opt_trace, Diagnostics_area, Protocol) are ≈ 9.6 % in all | `thd == current_thd` at entry with clone and dispatch, each admitted field with no writer on a path that reaches another thread's THD | **below the 3 % bar** (4 Oct census). Writers of another THD: KILL and the statement timer through `awake`, shutdown and offline mode, FLUSH STATUS (resets every THD's `status_var` under `LOCK_status` only), SET RESOURCE GROUP for a thread id, group replication's transaction contexts; readers: processlist, EXPLAIN FOR CONNECTION, SHOW STATUS, the performance schema visitors |


### Ledger: levers already measured (from `ideas.md` and the notes it cites)

| class | lever | standing | measurement |
|---|---|---|---|
| EA: object reached through a pointer argument | top-down facts; WP summaries; CLONE-ESC | closed by count | MySQL WP 0.14 %; top-down ≤ 0.14 % on all 5 apps (2 Oct) |
| EA: object from a call | named allocators (`-tsan-ea-alloc-fns`) | closed: no reach | no allocator wrapper carries malloc/noalias (missed census item 2) |
| EA: loaded slot escapes the slot | EA-CONTENTS | in the memcached best (V4) | — |
| DE: no cover at the same location | old same-location rule (S/A/Z) | UNSOUND, measurement only | S+A+Z 18-35 % of executed checks (3 Oct); proxy rule 0.1-1.1 %, not adopted |
| DE: unknown call between | DE across sync-free calls, IPA-DE | closed: subsumed by verified placement | missed census item 1 |
| DE: no cover on every path | all-paths dataflow, cycle cut | in the best + DE trees | DE-5 cycle cut ≈ 1 % |
| LO: mutex not recognised | LO-W | closed | Eraser ceiling ≤ 1.41 % |
| LO: object-relative locking | LO-OBJ-G | in the SQLite best (v5, spec v7) | — |
| LO: per-element locksets | PERELEM | parked (stopped at 1.67 %) | memcached ~2.3 % |
| STC: function context | STC-1..4, join-aware STC, N1-ST-WORKER | closed | oracle bounds ≈ 0; N1-ST-WORKER 0.00 % |
| SWMR: write-once globals | SWMR-G, per-field SWMR, address-in-initializer | closed by count | each < 3 % |
| thread roles / main-only code | thread-history census | closed: no static form | Redis 49 % main-only, not static |
| ownership hand-off | OWN-HANDOFF, OWN-CONN | not adopted / withdrawn (29 Sep: premise not asserted in code) | — |
| own stack | OWN-STACK oracle | parked | MySQL +5.3 % unsound ceiling, sound ≈ −1 % |
| MySQL statics | WP, top-down, ODR trust, InnoDB latch | closed by count | ≤ 0.14 %, latch 3.2 % |
| ranges and interceptors | LIBCALL, MEMINTR one-side, MEMINTR-INLINE | closed / not built | LIBCALL ≤ 0.8 %; MEMINTR-INLINE ≈ 0.5-1 % (3 Oct) |
| runtime placement | N1-SPLIT, N1-PAIR, FE-HOT/PM/LAZY, PGO-PLACE | closed by timing or argument | — |
| run-time phases | QUIET-THREADS (Redis phase guard) | in work: design audits 3-4 Oct, fixes written; the record run has no global reset, so the gain needs the background threads' attested start | quiet reach on the other four apps ≤ 0.4 % of checks (4 Oct) |
| deadlock detector | DD-COST | parked 29 Sep | — |

## 6. SQLite: what the old same-location rule removes, and what a sound rule could reach

The question (4 Oct): the old rule's S+A+Z buys 1.455 on create_drop_index_1 (DSL timing, 4 Oct), and A
lost no race. Which removals carry that, and does a sound transformation reach them?

- **Method.**
  - The removals: the DSL census's cover reports for S+A+Z on SQLite (AllOpt removal mode; `dsl/out/sqlite/rep-*`).
  - The weights: a counting run of create_drop_index_1 alone (H2, 4 Oct 02:58: 3.9 G plain checks, 41.8 % misses).
  - A location counts once; the 239 locations without a debug line are left out
    (the census's `dslclass.py`).
- **Result.** S+A+Z removes **17.8 %** of create_drop_index_1's executed checks in the best build: S 6.7, A 6.2,
  S+Z 3.1, A+Z 1.8 (S alone 8.1 %, A alone 8.0 %).

| class | functions (share of the subtest's executed checks) | ceiling | sound route |
|---|---|---|---|
| objects of the connection, under `db->mutex` | `sqlite3VdbeAddOp3` 1.46 and `sqlite3VdbeMakeReady` 0.21 (the VDBE under construction, linked into `db->pVdbe` at creation, so reachable from other threads only under `db->mutex`); `vdbeSorterCompareInt` 1.53 and `sqlite3VdbeSorterWrite` 1.05 (the statement's sorter); lookaside (`setupLookaside`, `sqlite3DbMallocRawNN`, `sqlite3DbNNFreeNN` 1.2) | ≈ 5.5 % | **LO-OBJ-G on `db->mutex`** (new spec entries; the sorter only if its worker threads are off in the build) |
| objects under BtShared's mutex | `sqlite3BtreeNext` 0.25, `sqlite3PagerWrite` 0.23, `btreeLockCarefully` 0.25, `sqlite3BtreeEnter` 0.22, `vdbeLeave` 0.23 | ≈ 1.2 % | LO-OBJ-G (spec v7 covers part of them; the rest are fields the spec does not name) |
| adjacent bytes and loops over the SQL text | `sqlite3GetToken` 1.00, `keywordCode` 0.38, `tokenExpr` 0.28 | ≈ 1.7 % | merging adjacent narrow accesses (DE-2R) and one range check per loop (DE-3R) |
| the parser's stack | `yy_reduce` 0.97 (the parser engine's `yyStack`, inside `sqlite3RunParser`'s stack frame) | ≈ 1.0 % | own stack (OWN-STACK, parked; it measured 4.9 % of SQLite's checks on stable-4, not on this subtest) |
| equal indexes | none found | 0 | DE-AV adds nothing here |
| the rest (VDBE registers in `sqlite3VdbeExec` 0.76, scattered) | | ≈ 8 % | mostly connection-owned registers (the first class's kind) |

- **Reading.** The bulk of the gain is objects owned by the connection, used under `db->mutex`: the VDBE being
  built, the sorter and lookaside.
  - These are neither BtShared's (which LO-OBJ-G already handles) nor fresh: the VDBE is published into
    `db->pVdbe` the moment it is created.
  - A sound route is LO-OBJ-G with spec entries for them, above the 3 % bar, so it is a build candidate.
  - Adjacent bytes and loops (DE-2R/3R) come to about 1.7 %, and the parser's stack to about 1 %; neither passes
    the bar alone.
  - These are shares of executed checks on one subtest; the timing gain also includes the other DSL arms' differences
    from the best build (no LO-OBJ-G in the DSL trees), so it is not a ceiling for any one class.

## Limits

- **Counting runtime.**
  - It counts every access the runtime sees, atomics included (TSan's atomics call the same access path). Locations
    without an explain-census line (atomics, a few without debug info) are reported apart.
  - It perturbs timing, so interleavings and hit rates are those of a slower run.
  - The H2 builds drop the inline hit test (Redis); the checked set and the hit condition are the shipped build's.
- **Join key.** Locations are joined on file basename, line and column across builds of the same sources. Collisions
  are counted per app: 0 on memcached, Redis and SQLite, 5 on MySQL.
- **Classes T1/T2/S** come from the 26 Sep oracle profiles of the older workloads. They are marked per row, and "-"
  where that profile had no check.
- **FFmpeg** is weighted by its 26 Sep profile, which used the record input, and has no miss counts.
- **Native time** is from stock-over-native legs; no best-over-native leg exists yet. A best/native ratio would need
  two legs and is not quoted.
- **perf** (`cycles:u`) leaves out kernel time, which matters for memcached's sendmsg and read.
- **MySQL** is the 60 s run minus the 1 s run (counts) and the 30 s run minus the 1 s run (perf). Start-up-only
  sites can come out slightly negative and are floored at 0.
