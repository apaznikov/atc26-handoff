# Hotspots: where the best configurations still pay, and why

State: 5 Oct 2026. The profiles are of 3-4 Oct: each app's best configuration of that day on root
`tsan-integcr-1738fdee35e7`, record workloads. The levers measured since are marked where they apply; memcached and
Redis have not been profiled again. The full per-site tables of 4 Oct (15 sites per app, every census column) are in
the history, commit ec0d58e. Results: `optimization-results.md`. Russian: `hotspots.ru.md` (not committed).

**Legend:** ✅ taken, in a configuration of record · ⏳ built, awaiting a ruling · ⏸ parked · ✖ closed (no sound
route, or below the bar) · ○ open, not measured. Shares are of the instrumented run's user cycles (Redis: of its main
thread M) unless they say "of checks", which means executed plain-access checks. "Quiet mode" is the quiet threads
(QUIET-THREADS: silent-thread mode, the Redis phase guard); names and aliases as in `optimization-results.md`.

## 1. At a glance

| app | where the cycles go | hottest site (3 Oct) | what took it | what is left |
|---|---|---|---|---|
| **memcached** | ranges and interceptors 31 %, checks 30 % (88 % of them miss), mutexes 25 % | `memset` of each response object, **16.6 %** | ✅ EVCONF-RANGES (+14-19 %), then ✅ EVCONF-ARGS (+7 %) and ✅ EVCONF-INTERCEPT (+6-8 %) | mutexes and plain checks, about a quarter each; ⏳ connection-field reads (EVCONF-FIELDS, +3 %); ✖ unlocked flag globals |
| **Redis**, main thread | ranges and interceptors 36 %, checks 28 % (97 % hit), function entry 10 % | the reply list's `memcpy`, **14.1 %** | ✅ quiet mode with the range skip, +26 % over the best | function entry (○ QUIET-FE); the allocator; the guard's own code (≈ 6 %) |
| **SQLite** | checks 51 % (flat: one site above 0.8 %), ranges 17 %, mutexes 7 % | none dominates; `memcmp` of record bytes 3.7 % | ✅ LO-OBJ-G, before this report | objects reached through pointer arguments (45 % of checks, no static route); ⏸ connection objects under `db->mutex`; ⏸ DE-AV |
| **MySQL** | checks 41 % (flat, top site 0.36 %), atomics and sync 15 %, function entry 12 % | none | — | 62.5 % of checks are on genuinely shared memory; ○ EA call-site census |
| **FFmpeg** | mjpeg: checks 58 %; copy: ranges and allocator 27 %; h264/h265: about 60 % in uninstrumented x264/x265 | mjpeg's codec contexts | ✅ DynSTC-RT + N1-ST, before this report | frozen since 3 Oct |

What paid: a range site that concentrates the cost, an unsound ceiling first (the site uninstrumented), then the
sound design and its audit. Plain checks never concentrate this way: no plain-check site exceeds 5.4 % of an app's
cycles, and on SQLite and MySQL none exceeds 1 %.

## 2. What this report turned into (4-5 Oct)

| hotspot (3 Oct) | lever | unsound ceiling | measured, sound | standing |
|---|---|---|---|---|
| memcached: `memset(resp, 0, sizeof(*resp))` in `resp_allocate`, 16.6 % | EVCONF-RANGES: EVCONF's guard on memory intrinsics over an owned object | `f` +14.1 % | `f` +13.9 %, `a` +18.5 % | ✅ |
| memcached: MurmurHash3's reads of the request key, 5.4 % (26.6 % of checks, all miss) | EVCONF-ARGS: a clone of the hash with the key reads unchecked while the guard holds | `f` +8.2 %, `a` +5.7 % | `f` +7.2 %, `a` +7.3 % (≈ 83 % of the key reads) | ✅ |
| memcached: `memchr`, `strlen` and the request-key side of `bcmp` on the read buffer, ≈ 4.7 % | EVCONF-INTERCEPT: the guard around those libc calls | `f` +6.5 % | `f` +5.7 %, `a` +7.5 % | ✅ |
| memcached: reads of the connection's own fields, ≥ 14 % of checks (77 % miss) | EVCONF-FIELDS: the owner's reads unchecked, writes checked | `f` +3.6 % | `f` +2.9 %, `a` +3.6 % | ⏳ the wider P-X86-FD |
| Redis: the reply-list `memcpy` (14.1 %) and `initEntry`'s `memset` (4.8 %) | quiet mode's range skip | — | +3.6…+9.9 % on top of quiet mode | ✅ |
| memcached: `settings`, `expanding`, `hashpower` read without a lock, ≈ 5 % (5.5 % of checks) | — | `f` +4.5 % | — | ✖ admin-written or a genuine benign race |
| SQLite: `memcmp` (3.7 %) and VDBE `memcpy` (2.0 %) on lock-owned bytes | LO-OBJ-RANGES | — | — | ✖ reaches neither site |
| SQLite: the record comparison, ≤ 4.4 % | LO-OBJ-ARGS | +2.3 %, inside the A/A | — | ✖ |
| SQLite: the connection's objects under `db->mutex`, ≈ 5.5 % of create_drop_index_1's checks | LO-OBJ-G with a second lock | `a` +5.8 % | — | ⏸ needs two lock types and a ruling on how the lock is asserted |
| MySQL: a THD's fields read by its own thread, ≤ 2.35 % of checks | owner reads | — | — | ✖ below the 3 % bar |
| Redis: the I/O threads, 88 % of all cycles | — | — | — | ✖ 98.5 % of it is a spin, no checks |
| objects one thread touches (T1), 17-53 % of checks | a run-time owner tag | — | — | ✖ unsound: a skipped access leaves no record to race with |

## 3. Where the time goes

`perf record -e cycles:u --call-graph=lbr`, 3 runs per app, 3 Oct. Columns are shares of the app's cycles. "Checks"
splits into N1's inline hits, hits inside a runtime entry, and misses (the rest of the entry and the race check).

| app (threads) | checks (inline hit / entry hit / miss) | function entry | ranges + interceptors | atomics + sync | runtime internals + deadlock detector | program | other |
|---|---|---|---|---|---|---|---|
| Redis, main thread (12 % of cycles) | 27.8 (22.3 / 1.2 / 4.3) | 10.3 | **36.4** | 1.7 | 0.2 | 21.4 | 2.2 |
| Redis, I/O threads (88 %) | 0.0 | 0.0 | 0.1 | **98.5** (spin) | 0.0 | 1.4 | 0.0 |
| memcached (workers) | 29.5 (0.2 / 15.4 / 13.9) | 1.2 | **31.3** | **24.8** | 2.1 | 7.5 | 3.6 |
| SQLite | **50.6** (0.2 / 31.5 / 18.9) | 3.3 | 17.4 | 7.3 | 1.1 | 16.5 | 3.8 |
| MySQL, timed phase | **40.6** (0.2 / 22.6 / 17.8) | 12.1 | 7.3 | 14.8 | 0.6 | 23.0 | 1.6 |
| FFmpeg h264 | 17.1 (9.6 / 0.1 / 7.4) | 1.2 | 2.4 | 8.4 | 0.5 | 11.6 | 58.9 (x264) |
| FFmpeg h265 | 9.4 (5.1 / 0.1 / 4.2) | 0.7 | 21.1 | 1.1 | 0.1 | 6.3 | 61.3 (x265) |
| FFmpeg mjpeg | **58.4** (27.2 / 1.2 / 30.0) | 2.3 | 4.8 | 1.3 | 0.4 | 32.2 | 0.5 |
| FFmpeg copy | 3.5 (3.1 / 0.4 / 0.0) | 3.9 | **26.8** | 2.0 | 0.5 | 55.2 | 8.1 |

- **Hit rates differ by an order of magnitude** (counting runs): memcached 12.2 %, MySQL 54.6 %, SQLite 71.3 %,
  Redis's main thread 96.9 %. memcached takes and releases mutexes on every request, each a new epoch, so a check
  rarely finds its own record; there every removed check saves a slow path. On Redis the cost is the number of
  checks, not the misses.
- **Stock TSan over native** (one leg each): Redis 6.0×, memcached 6.7×, SQLite 9.2× (stress2 4.5×,
  create_drop_index_1 16-21×), MySQL 7.5×, FFmpeg 2.8× (copy 5.5-5.9×, mjpeg 6.0-6.2×, h264 1.3×, h265 1.4×).
  Instrumentation is therefore 64-89 % of a run (FFmpeg's encoders 20-30 %), and a category's share of the overhead
  is its share above divided by that fraction.

## 4. Per application

Each table lists the sites that cost most, plain checks and ranges together, with the refusal that keeps each one
checked. Classes from the 26 Sep oracles: T1 one thread, T2 consistently locked or written before publication (both
a static imprecision), S genuinely shared.

### memcached (best of 3 Oct: EVCONF + SWMR-ROOTS + EA-CONTENTS)

| cycles | site | object | why still checked | now |
|---|---|---|---|---|
| 16.6 % | `memset` in `resp_allocate` (memcached.c:1085) | the worker's own response object (1,176 bytes) | EVCONF covered plain accesses only | ✅ EVCONF-RANGES |
| 5.4 % | `getblock32` in MurmurHash3 (murmur3_hash.c:50) | the request key in the connection's read buffer | EA: pointer argument; EVCONF did not follow the call | ✅ EVCONF-ARGS |
| ≈ 5 % | `assoc_find`, `process_get_command`, `do_item_get`, `conn_set_state` | the globals `expanding`, `settings.*`, `hashpower` | LO: no lock held; SWMR: the address escapes | ✖ |
| ≈ 4.7 % | `memchr` 2.1, `strlen` 1.6, `bcmp` 1.0 (the key half of 2.0) | the read buffer | interceptor ranges were outside EVCONF | ✅ EVCONF-INTERCEPT |
| 2.7 % | `main` 0.99, `assoc_find` 0.79, `complete_nread_ascii` 0.58, `drive_machine` 0.30 | — | no debug line, so the census cannot place them | ○ SITE-ID census |
| 1.8 % | `read` into the read buffer | the read buffer | the interceptor also carries the descriptor's acquire | ○ |

- **Mutexes, 24.8 %:** `item_lock` 7.5 (the global hash-partitioned lock array: shared, real synchronisation) and the
  worker's own stats mutex ≈ 6.4 (taken by its owner on every response, rarely by the aggregator). No pass can remove
  a mutex operation; a cheaper owner-to-owner path in the runtime is ⏸ with the runtime track. The deadlock detector
  is 2.0 %.
- **After the 4-5 Oct levers** mutexes and plain checks are about a quarter of the cycles each *(by subtraction, not
  re-profiled)*.

### Redis, main thread (best of 3 Oct: FE-INL + N1)

| cycles | site | object | why still checked | now |
|---|---|---|---|---|
| 14.1 % | `memcpy` in `_addReplyProtoToList` (networking.c:361) | the client's reply block: M fills it, the I/O threads drain it | handed between threads by phase: no static fact | ✅ quiet range skip |
| 8.2 % (+0.9) | `malloc`, `free`, `realloc` through zmalloc | — | the allocator's bookkeeping, not a check | stays |
| 4.8 % | `memset` in `initEntry` (quicklist.c:1300) | 16 bytes of the caller's stack | EA: pointer argument | ✅ quiet range skip |
| 4.8 % | `quicklistNext`, `listTypeNext` (the LRANGE iterator) | a `quicklistEntry` on the caller's stack (T1) | EA: pointer argument | ✅ quiet mode |
| 4.4 % | `_addReplyProtoToList` reads the tail block (3.08) and writes its `used` (1.35) | the reply list (S, T2) | EA: pointer argument; LO, SWMR: not a global | ✅ quiet mode |
| 10.3 % | function entry and exit | — | FE-INL records every frame | ○ QUIET-FE |

- **What quiet mode leaves:** it reads 1.256-1.314 over the best on Intel, against an unsound ceiling of 1.365 for
  skipping all of M's plain checks. The rest is function entry, the allocator, and the guard's own compiled tests
  (about 6 % of M's cycles; folding them is ⏸).
- **The I/O threads** spend 98.5 % of their cycles spinning on `io_threads_pending`; nothing there to take.

### SQLite (best: LO-OBJ-G v5 + spec v7)

| cycles | site | object | why still checked | now |
|---|---|---|---|---|
| 3.7 % | `memcmp` in `vdbeRecordCompareString` | record bytes on a b-tree page against the caller's key | the page side arrives through a function pointer | ✖ LO-OBJ-ARGS |
| 2.0 % | VDBE register `memcpy` in `sqlite3VdbeExec` | the statement's registers | `sqlite3_value_dup` reads them without `db->mutex` | ✖ LO-OBJ-RANGES |
| 1.9 % | `strHash` 0.77, `sqlite3StrICmp` 0.71 + 0.45 | identifiers from the SQL text and the schema (T2) | LO, SWMR: not a global | ⏸ part of the `db->mutex` objects |
| 1.6 % | `sqlite3_str_vappendf` | — | no debug line | ○ |
| 0.7 % | `sqlite3_mutex_enter`/`leave` reading `sqlite3Config` | a global written at start-up (T2) | its address is in an initializer | ○ SWMR-ROOTS on SQLite |
| 7.3 % | `pthreadMutexEnter`/`Leave` | shared cache, page cache, memory | real synchronisation | stays |

- **Flat.** T1 is 53 % of the checks (pointer arguments 45 %, objects from a call 7 %), T2 21 %, all on non-globals:
  the parser's stack and engine, the VDBE's registers, b-tree page content. Those under BtShared's mutex are
  LO-OBJ-G's already.
- **What the unsound same-location rule removes on create_drop_index_1** (17.8 % of its checks), by class:

  | class | share of the subtest's checks | sound route | standing |
  |---|---|---|---|
  | the connection's objects under `db->mutex`: the VDBE under construction, the sorter, lookaside | ≈ 5.5 % | LO-OBJ-G with spec entries for them | ⏸ (ceiling +5.8 %) |
  | fields under BtShared's mutex that spec v7 does not name | ≈ 1.2 % | spec lines | ○ |
  | adjacent bytes and loops over the SQL text | ≈ 1.7 % | DE-2R, DE-3R | below the bar alone |
  | the parser's stack (`yy_reduce`) | ≈ 1.0 % | own stack | ⏸ |
  | equal indexes | 0 | DE-AV | — |
  | the rest, mostly connection-owned registers | ≈ 8 % | as the first row | — |

### MySQL (best: FE-INL, Release)

Flat: the top site is 0.36 % of the cycles. 62.5 % of the checks touch genuinely shared memory.
- InnoDB's mutex implementation (`ut0mutex.h`, `ib0mutex.h`) and buffer-pool page reads (`mach_read_from_1/2`): S,
  real.
- System variables read without a lock: `srv_n_spin_wait_rounds` 0.36, the spin multiplier 0.19, `innodb_hton_ptr`
  0.15, `srv_read_only_mode` 0.14 (0.84 % in all). Those written by SET GLOBAL have no sound route, as memcached's
  `settings`; init-once ones could take a SWMR-ROOTS rule (○).
- `std::thread::id` built on the stack: 0.46 %, S by the oracle.
- Atomics 14.8 % (`std::atomic` loads 6.5): 74 % of the atomic entries are seq_cst, so an inline relaxed-atomic test
  reaches ≤ 0.65 % (✖).
- Function entry 12.1 %: FE-INL is in the best on AMD and loses 3 % on Intel.
- The EA call-site census (counting run done: 80.4 G checks in the timed phase, 34.9 % on the accessing thread's own
  stack) has not been joined with the IR yet (○).

### FFmpeg (frozen since 3 Oct; for the record)

The cost is in mjpeg (checks 58.4 %: codec contexts reached through pointer arguments, T1) and in h264 decoding
(CABAC, per-slice state, T2). h264 and h265 run about 60 % of their cycles in uninstrumented libx264/libx265, whose
intercepted libc calls are the "ranges" there. The stream copy pays the allocator and option parsing at start-up.

## 5. Blocker classes across applications

Rows 1-5 are shares of plain checks; rows 6-8 shares of cycles.

| class | memcached | Redis main | SQLite | MySQL | FFmpeg | standing |
|---|---|---|---|---|---|---|
| EA: an object reached through a pointer argument (T1) | 16.9 % (+11.9 % unclassified) | 37.2 % | 45.2 % | 17.0 % | 20.9 % | static routes ✖ (cross-unit facts ≤ 0.14 % everywhere); run-time: ✅ quiet mode (Redis), ⏸ own stack, ○ call-site census |
| EA: an object returned by a call (T1) | 0.3 % | 2.2 % | 6.9 % | 2.5 % | 0 | ✖ no allocator wrapper carries `malloc` |
| LO and SWMR judge only globals (T2, not a global) | 0.9 % | 10.8 % | 17.6 % | 6.7 % | 42.5 % | ✅ LO-OBJ-G (SQLite's BtShared); ⏸ its `db->mutex` extension |
| globals read without a lock, or whose address escapes (T2 globals) | 7.7 % | 0.2 % | 1.7 % | 2.7 % | 0 | ✖ memcached (admin-written, or a benign race); ○ SWMR-ROOTS on SQLite and MySQL |
| genuinely shared, no cover at the same location (S) | 52.8 % | 31.0 % | 15.7 % | 57.8 % | 21.9 % | real races are possible here; the old same-location rule is unsound; ⏸ DE-AV |
| ranges and interceptors | 31.3 % | 36.4 % | 17.4 % | 7.3 % | copy 26.8 %, h265 21.1 % | ✅ memcached (RANGES, INTERCEPT); ✅ Redis (quiet range skip); ✖ SQLite; the rest is allocator or uninstrumented callers |
| mutexes and atomics | 24.8 % | 1.7 % | 7.3 % | 14.8 % | h264 8.4 % (x264) | real synchronisation; ⏸ owner-to-owner mutex path (runtime track); ✖ N1-ATOMIC |
| function entry and exit | 1.2 % | 10.3 % | 3.3 % | 12.1 % | ≤ 2.3 % | FE-INL in the best; ○ QUIET-FE |

## Method and limits

- **Time split:** `perf record -e cycles:u --call-graph=lbr`, 3 runs per app; each runtime entry's cycles go to its
  first instrumented caller. MySQL's timed phase is a 30 s run minus a 1 s run. Kernel time (memcached's `sendmsg`
  and `read`) is not in `cycles:u`.
- **Counts:** one record-workload run per app with a counting runtime (measure/hotspots `5bd1070eef6f`): the shipped
  build's checked set and hit condition; per caller pc the executed checks, the misses and the class of the missed
  address. Redis's counting build drops the inline hit test. MySQL is a 60 s run minus a 1 s run. FFmpeg has no
  counting run: its 26 Sep profile of the same input gives executed checks only. The counting runtime counts atomics
  too (reported apart) and slows the run, so hit rates are those of a slower run.
- **Why a site is checked:** `-tsan-explain-missed` with each app's best flags over its pre-TSan IR, giving per load
  and store its status and the first refusal of each analysis.
- **Classes T1/T2/S** come from the 26 Sep oracle profiles of older workloads; "-" where that profile had no check.
- **Join:** file, line and column across builds of the same sources; collisions 0, except 5 on MySQL. A site at line 0
  has no debug line and is reported unattributed.
