# Hotspots: where the best configurations still pay, and why

State 7 Oct 2026; profiles of 3-4 Oct (root `tsan-integcr-1738fdee35e7`, record workloads), not repeated since. Longer earlier versions: 9666281, ec0d58e. Results: `optimization-results.md`.

## 1. At a glance

- **Kind** of a lever: S static · R run-time checked, no annotation · G generated from the program's assertions · A hand-written annotation (A+R: run-time guarded) · RT runtime-only.
- **Standing**: ✅ in a configuration of record · ⏳ awaiting a ruling · ⏸ parked · ✖ closed · ○ open.
- **Shares** are of the instrumented run's user cycles (Redis: of its main thread M); "of checks" = executed plain-access checks.

| app | best now (over stock TSan) | where the cycles went (3 Oct) | hottest site | what took it | what is left |
|---|---|---|---|---|---|
| **memcached** | 1.02× S; 1.76× R (EVCONF-CHECKED); 1.81× A+R | ranges and interceptors 31 %, checks 30 % (88 % miss), mutexes 25 % | `memset` of each response object, **16.6 %** | ✅ EVCONF-RANGES, -ARGS, -INTERCEPT (A+R) | mutexes and plain checks, a quarter each; ⏳ EVCONF-FIELDS (A+R); ○ EVCONF-CHECKED (R); ✖ unlocked flag globals |
| **Redis**, main thread | 1.54× R | ranges and interceptors 36 %, checks 28 % (97 % hit), function entry 10 % | the reply list's `memcpy`, **14.1 %** | ✅ quiet threads with the range skip (R) | function entry (○ QUIET-FE, R); the allocator; the guard's own code (≈ 6 %) |
| **SQLite** | 1.16× G+R; 1.25× A+R | checks 51 % (no site above 0.8 %), ranges 17 %, mutexes 7 % | `memcmp` of record bytes, 3.7 % | ✅ LO-OBJ-G (G+R, A+R) | pointer-argument objects (45 % of checks); ⏸ `db->mutex` objects; ⏸ DE-AV (R) |
| **MySQL** | 1.13× S | checks 41 % (top site 0.36 %), atomics and sync 15 %, function entry 12 % | none | ✅ FE-INL (S) | 62.5 % of checks on genuinely shared memory; ○ EA call-site census |
| **FFmpeg** | 1.29× R | mjpeg: checks 58 %; copy: ranges and allocator 27 %; h264/h265: ≈ 60 % in uninstrumented x264/x265 | mjpeg's codec contexts | ✅ DynSTC-RT, N1-ST (R) | frozen since 3 Oct |

- No plain-check site exceeds 5.4 % of an app's cycles (on SQLite and MySQL none exceeds 1 %); what paid were range sites.

## 2. What this report turned into (4-5 Oct)

| hotspot (3 Oct) | lever (kind) | unsound ceiling | measured, sound | standing |
|---|---|---|---|---|
| memcached: `memset` in `resp_allocate`, 16.6 % | EVCONF-RANGES (A+R) | `f` +14.1 % | `f` +13.9 %, `a` +18.5 % | ✅ |
| memcached: MurmurHash3's key reads, 5.4 % (26.6 % of checks, all miss) | EVCONF-ARGS (A+R) | `f` +8.2 %, `a` +5.7 % | `f` +7.2 %, `a` +7.3 % | ✅ |
| memcached: `memchr`, `strlen`, key side of `bcmp`, ≈ 4.7 % | EVCONF-INTERCEPT (A+R) | `f` +6.5 % | `f` +5.7 %, `a` +7.5 % | ✅ |
| memcached: the connection's own fields, ≥ 14 % of checks (77 % miss) | EVCONF-FIELDS (A+R) | `f` +3.6 % | `f` +2.9 %, `a` +3.6 % | ⏳ wider P-X86-FD |
| Redis: reply-list `memcpy` (14.1 %), `initEntry`'s `memset` (4.8 %) | quiet range skip (R) | — | +3.6…+9.9 % on top of quiet mode | ✅ |
| memcached: `settings`, `expanding`, `hashpower` unlocked, ≈ 5 % | — | `f` +4.5 % | — | ✖ admin-written or a benign race |
| SQLite: `memcmp` (3.7 %), VDBE `memcpy` (2.0 %) | LO-OBJ-RANGES (A+R) | — | — | ✖ reaches neither site |
| SQLite: the record comparison, ≤ 4.4 % | LO-OBJ-ARGS (A+R) | +2.3 %, inside the A/A | — | ✖ |
| SQLite: `db->mutex` objects, ≈ 5.5 % of create_drop_index_1's checks | LO-OBJ-G, second lock (A+R) | `a` +5.8 % | — | ⏸ |
| MySQL: a THD's fields read by its thread, ≤ 2.35 % of checks | owner reads (S) | — | — | ✖ below the bar |
| Redis: the I/O threads, 88 % of all cycles | — | — | — | ✖ 98.5 % is a spin |
| T1 objects, 17-53 % of checks | run-time owner tag | — | — | ✖ unsound |

## 3. Where the time goes (3 Oct, shares of cycles)

| app (threads) | checks (inline hit / entry hit / miss) | function entry | ranges + interceptors | atomics + sync | runtime + deadlock detector | program | other |
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

- Hit rates: memcached 12.2 %, MySQL 54.6 %, SQLite 71.3 %, Redis main thread 96.9 %.
- Stock TSan over native: Redis 6.0×, memcached 6.7×, SQLite 9.2× (stress2 4.5×, create_drop_index_1 16-21×), MySQL 7.5×, FFmpeg 2.8× (copy 5.5-5.9×, mjpeg 6.0-6.2×, h264 1.3×, h265 1.4×).

## 4. Per application

### memcached (profiled on EVCONF + SWMR-ROOTS + EA-CONTENTS)

| cycles | site | object | why still checked | now |
|---|---|---|---|---|
| 16.6 % | `memset` in `resp_allocate` (memcached.c:1085) | the worker's own response object | EVCONF covered plain accesses only | ✅ EVCONF-RANGES |
| 5.4 % | `getblock32` in MurmurHash3 | the request key in the read buffer | EA: pointer argument | ✅ EVCONF-ARGS |
| ≈ 5 % | `assoc_find`, `process_get_command`, `do_item_get`, `conn_set_state` | `expanding`, `settings.*`, `hashpower` | LO: no lock; SWMR: address escapes | ✖ |
| ≈ 4.7 % | `memchr` 2.1, `strlen` 1.6, `bcmp` 1.0 | the read buffer | interceptor ranges outside EVCONF | ✅ EVCONF-INTERCEPT |
| 2.7 % | `main`, `assoc_find`, `complete_nread_ascii`, `drive_machine` | — | no debug line | ○ SITE-ID census |
| 1.8 % | `read` into the read buffer | the read buffer | carries the descriptor's acquire | ○ |
| 7.5 % | `item_lock` | the hash-partitioned lock array | real synchronisation | stays |
| ≈ 6.4 % | the worker's stats mutex | taken by its owner on every response | real synchronisation | ⏸ owner-to-owner path (RT) |

### Redis, main thread (profiled on FE-INL + N1)

| cycles | site | object | why still checked | now |
|---|---|---|---|---|
| 14.1 % | `memcpy` in `_addReplyProtoToList` (networking.c:361) | the reply block, drained by the I/O threads | handed over by phase | ✅ quiet range skip |
| 8.2 % (+0.9) | `malloc`, `free`, `realloc` via zmalloc | — | allocator bookkeeping | stays |
| 4.8 % | `memset` in `initEntry` (quicklist.c:1300) | 16 bytes of the caller's stack | EA: pointer argument | ✅ quiet range skip |
| 4.8 % | `quicklistNext`, `listTypeNext` | a `quicklistEntry` on the caller's stack (T1) | EA: pointer argument | ✅ quiet threads |
| 4.4 % | `_addReplyProtoToList`, the tail block | the reply list (S, T2) | EA: pointer argument; not a global | ✅ quiet threads |
| 10.3 % | function entry and exit | — | FE-INL records every frame | ○ QUIET-FE |

- Quiet threads read 1.256-1.314 over FE-INL + N1 on Intel (5 Oct) against an unsound ceiling of 1.365.

### SQLite (profiled on LO-OBJ-G v5 + spec v7)

| cycles | site | object | why still checked | now |
|---|---|---|---|---|
| 3.7 % | `memcmp` in `vdbeRecordCompareString` | page record bytes vs the caller's key | page side via a function pointer | ✖ LO-OBJ-ARGS |
| 2.0 % | VDBE register `memcpy` | the statement's registers | read without `db->mutex` | ✖ LO-OBJ-RANGES |
| 1.9 % | `strHash`, `sqlite3StrICmp` | SQL-text and schema identifiers (T2) | not a global | ⏸ `db->mutex` objects |
| 1.6 % | `sqlite3_str_vappendf` | — | no debug line | ○ |
| 0.7 % | `sqlite3_mutex_enter`/`leave` reading `sqlite3Config` | start-up global (T2) | address in an initializer | ✖ SWMR-ROOTS (closed 5 Oct) |
| 7.3 % | `pthreadMutexEnter`/`Leave` | shared cache, page cache, memory | real synchronisation | stays |

What the unsound same-location rule removes on create_drop_index_1 (17.8 % of its checks):

| class | share of checks | sound route | standing |
|---|---|---|---|
| connection objects under `db->mutex` (VDBE, sorter, lookaside) | ≈ 5.5 % | LO-OBJ-G spec entries | ⏸ ceiling +5.8 % |
| BtShared fields spec v7 does not name | ≈ 1.2 % | spec lines | ○ |
| adjacent bytes, loops over the SQL text | ≈ 1.7 % | DE-2R, DE-3R | below the bar |
| the parser's stack (`yy_reduce`) | ≈ 1.0 % | own stack | ⏸ |
| the rest, mostly connection-owned registers | ≈ 8 % | as the first row | — |

### MySQL (FE-INL, Release)

| cycles | site or class | why still checked | now |
|---|---|---|---|
| — | InnoDB mutexes, buffer-pool page reads | S, real | stays |
| 0.84 % | unlocked system variables (`srv_n_spin_wait_rounds` 0.36 …) | SET GLOBAL writes them | ✖ (init-once ones: SWMR-ROOTS closed) |
| 0.46 % | `std::thread::id` on the stack | S by the oracle | stays |
| 14.8 % | atomics (`std::atomic` loads 6.5) | 74 % seq_cst | ✖ N1-ATOMIC (≤ 0.65 %) |
| 12.1 % | function entry | — | ✅ FE-INL on AMD (Intel −3 %) |
| — | 34.9 % of 80.4 G checks on the thread's own stack | not joined with the IR | ○ EA call-site census |

### FFmpeg (frozen since 3 Oct)

- mjpeg: checks 58.4 % on codec contexts via pointer arguments (T1); h264: CABAC and per-slice state (T2); copy: allocator and option parsing.

## 5. Blocker classes across applications

- **Classes** (26 Sep oracles): T1 one thread, T2 consistently locked or written before publication, S genuinely shared.

| class | memcached | Redis main | SQLite | MySQL | FFmpeg | standing |
|---|---|---|---|---|---|---|
| EA: object via a pointer argument (T1), of checks | 16.9 % (+11.9 % unclassified) | 37.2 % | 45.2 % | 17.0 % | 20.9 % | static ✖ (cross-unit facts ≤ 0.14 %); ✅ quiet threads R; ⏸ own stack R; ○ call-site census |
| EA: object returned by a call (T1), of checks | 0.3 % | 2.2 % | 6.9 % | 2.5 % | 0 | ✖ no wrapper carries `malloc` |
| LO, SWMR judge only globals (T2), of checks | 0.9 % | 10.8 % | 17.6 % | 6.7 % | 42.5 % | ✅ LO-OBJ-G G+R / A+R; ⏸ `db->mutex` |
| unlocked or escaping globals (T2), of checks | 7.7 % | 0.2 % | 1.7 % | 2.7 % | 0 | ✖ memcached; ✖ SWMR-ROOTS elsewhere (closed 5 Oct) |
| genuinely shared, no exact cover (S), of checks | 52.8 % | 31.0 % | 15.7 % | 57.8 % | 21.9 % | ⏸ DE-AV R |
| ranges and interceptors, of cycles | 31.3 % | 36.4 % | 17.4 % | 7.3 % | copy 26.8 %, h265 21.1 % | ✅ memcached A+R; ✅ Redis R; ✖ SQLite |
| mutexes and atomics, of cycles | 24.8 % | 1.7 % | 7.3 % | 14.8 % | h264 8.4 % | ⏸ owner-to-owner path RT; ✖ N1-ATOMIC |
| function entry and exit, of cycles | 1.2 % | 10.3 % | 3.3 % | 12.1 % | ≤ 2.3 % | ✅ FE-INL S; ○ QUIET-FE R |

## Method

- Time: `perf record -e cycles:u --call-graph=lbr`, 3 runs per app; kernel time not included.
- Counts: one record-workload run per app with a counting runtime (`5bd1070eef6f`); FFmpeg from its 26 Sep profile.
- Why checked: `-tsan-explain-missed` over each app's pre-TSan IR; sites joined by file, line and column.
