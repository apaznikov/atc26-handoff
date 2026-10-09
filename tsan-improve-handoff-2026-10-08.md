# tsan-improve handoff, 8 Oct 2026 (focs time): parking for an account move

Read this first when resuming the coordinator role. Token mode: ECONOMY (`token-modes.md`). Results of record:
`optimization-results.md`; premises: `soundness-fixes.md` §6; hotspots: `hotspots.md`. All three are tracked on main
and published with `env -u GITHUB_TOKEN FILE=... MSG=... BODY=... bash publish-results.sh`. Ideas and status:
`ideas.md` (row SQGEN-AUTO carries the SQLite story). Tracker: `experiments.md`. All under /home/alexey/atc26-handoff/.
Previous handoff: `tsan-improve-handoff-2026-10-05.md`.

## Priority set on 8 Oct: everything on SQLite

The user's instruction (8 Oct, afternoon): all lanes focus on SQLite; the two-archive runtime plan is postponed.
Goal: move SQLite out of hand annotations, i.e. column (4) → (3) and (3) → (2) of Table 1, as fast as possible.

### Table 1 as published (8 Oct)

| app | (1) static | (2) run-time checked | (3) generated from assertions | (4) annotations |
|---|---|---|---|---|
| SQLite (stress2) | ≈ 1.0× | — (path B in progress) | 1.19× (gen9b; zgb4) | 1.25× (v7) |
| FFmpeg | — | 1.29× (1.085× of it is dynstc_rt) | — | — |
| Redis | 1.10× | 1.54× | — | 1.40× over upstream |
| MySQL | 1.13× over upstream | — | — | — |
| memcached (V4) | 1.02× | 1.80× over upstream (EVCONF-CHECKED + marker gate Delta 1; kmg) | — | 1.81× |

Base = stock TSan of the same compiler unless the cell says "over upstream" (upstream = `tsan-pristine-up59b2-0f1aed148576`).

### SQLite, column (4) → (3): derived parser params (nearly done)

- sqgen19 derives the three `param btreeParseCellPtr{,Index,NoPayload} pCell owner pPage` lines without profile or hand
  lines. Same directive set as the diagnostic zgBp, measured +4.8 % over gen9b (zgp4, root 8345a).
- P-SUBARRAY (sub-arrays of one allocation stay in range) is replaced by a run-time check, not a premise: `carve`
  directives (generator) plus `__tsan_carve_*` (pass + runtime). Branch experiment/carve-check, head a5307a41f662
  (per-thread runtime, address-based reads, copy straddle rule, both allocators). Census on stress2: 13.1 M calls,
  458 M flag tests, 0 voids, 0 races.
- Audits: A72 (sqgen19), A73 with Delta 1 and Delta 2 (gen19e + carve). The SQLite spec is SOUND WITH CONDITIONS;
  the general rules remain UNSOUND (fresh liars that SQLite does not reach).
- R1-R5 run-time reproducers are done (`/extra/alexey/de-improve/sqauto/repro19/`).
  - Stock reports 5/5; gen19 reports 0/5, so the elision fires. Statically R1 and R4 withdraw the param, R2 keeps it.
  - **R3 (the second file's temp space under the first file's lock) is KEPT by sqgen19e.** Owner equality at the
    temp-space site is assumed, not derived. It is the top item for sqgen19f. If SQLite's real sites lose the params
    after the fix, column (3)'s figure changes.
  - R5 (carve) voids at a write out of its region.
- Open before quoting:
  - R3 decision (8 Oct, coordinator): it is premise B2's per-type gap, which gen9b's quoted figure already rests on. B2 is now its own row in soundness-fixes §6; R3 is recorded as B2's counterexample fixture. Page provenance ("(A)", ~1 day) would remove B2 for the params; it is an option after path B.
  - sqgen19f (tsan-dev-3, ETA ~3 h from 17:00): then A73's liars C1, the D class, A1 and R8a; the per-source carvefn closure; the
    r0 privacy proof (needed by the per-thread runtime); the inttoptr refusal;
  - audit A74 (sqgen19f + lockcfg.py).
- Timing set queued on apollo A after zrc4: zgBe2 (gen9b) | zg19 (gen19, no carve) | zg19c (gen19e + carve) | stock,
  one root tsan-carve-a5307a41f662. zg19c/zgBe2 is the column-(3) gain; zg19c/zg19 is carve's cost.
- Last hand input: the lock field. The user chose name-free (8 Oct). tsan-dev-3's lock filter
  (`sqauto/lockcfg/lockcfg.py` f843c4d2273d6a96, output mt.locks) derives the allocated mutexes from threadtest3's
  configuration without assertions. It keeps BtShared, unixInodeInfo, unixShmNode, MemStore and PGroup (PGroup is
  kept fail-closed; dropping it needs an init-order premise, half a day, only if the census says pcache1 guards cost).

### SQLite, column (3) → (2): path B (in progress)

- What it is: no assertions. Candidates = fields accessed under their owner's lock, by dataflow. Soundness comes from
  run time:
  - an unheld access checks happens-before from the lock's last release and that there is no holder, then records
    a dirty epoch per owner;
  - the first held test after an acquire checks dirty ≤ the clock;
  - a violation voids the run (exit 67).
  Owner bind, orphan records, a retire tomb on mutex destroy, and an ignore_sync void are all in.
- Root `tsan-lob-fbe442ec2e2c` (runtime 5ae6fe82f07e + pass f808bd6a09ac + e2e twins). Gates: IR, runtime lit,
  full tsan, check-tsan x12 12/12, go-check. Audit A71: the core is SOUND WITH CONDITIONS, the as-specified design
  was UNSOUND, and its items are fixed.
- Ceiling (unsound, U): zgu4 on the zgbu root, zgBu/zgBe 1.149, so B is worth building. IntegrityCk heap (zgBi) 1.005,
  so G-EA is dropped. zgf4 (B's field-only ceiling with no gen9b spec, zgF0/zg00) is running on A, ~17:15.
- Remaining:
  - C-COVER at access level, which needs allocation-site points-to (tsan-dev, ~1 day). MemPage/BtCursor likely
    drop unless the Vdbe/pcache frees and the pcache memset get directives;
  - the multi-lock pass for the five locks (moved to tsan-dev-2);
  - a B spec from the generator, then the void smoke (tsan-exp's runner `cov1-2026-09-27/p13/sqlite_void_smoke.sh`),
    preservation, a delta audit and timing.

### Root question (open)

- With identical flags, SQLite's LO-OBJ-G line reads 1.134 on the e52d line against 1.175-1.194 on 8345a (cross-leg).
  The compiled code is identical (texttree 3/3); only the runtime and layout differ. focs perf shows ~1 % at most.
- Same-leg check zrc4 (zqs | e52d stock | zgB | zgBe | A/A) is queued on apollo A after zgf4.

## Postponed by the user (8 Oct): the two-archive runtime plan

- Decided first: a standard archive (e52d's rtl byte for byte) plus `clang_rt.tsan_checked.a` (EVCONF-CHECKED with
  marker gate Delta 1, path B and carve), with `-fsanitize-thread-runtime=checked` and a strong reference that fails
  the link. Then paused for SQLite.
- tsan-dev-2 parks it at a clean point, with a state note in `evconf-checked.md` §15.
- Why: marker code in a shared runtime cost either stock memcached (Delta 1: 0.960 over upstream) or the EVCONF line
  (Delta 2: 1.737). Legs kmg, kmg2, kmg3 and kmg4 are in `experiments.md`; audits A70g, A70h (with d3/d4) and A70i.
- Merge branch experiment/evconf-e52d-merge (EVCONF + Delta 4 + path B + carve on e52d): gated and inert for
  non-users, apart from +2 instructions per free. Not frozen.

## Lanes at parking

| lane | doing |
|---|---|
| tsan-dev | C-COVER (R1-R5 done); EVCONF label determinism after that (postponed) |
| tsan-dev-2 | parking the two-archive work, then path B's multi-lock pass |
| tsan-dev-3 | sqgen19f (A73 items, carvefn closure, r0 privacy, inttoptr) |
| tsan-exp | apollo A: zgf4 → zrc4 → column-(3) set; B idle; memcached legs paused |

Hosts: focs (builds; no timed legs, the hien_* waiters rewritten to pause only for an announced FOCS_LEG), apollo
(timed legs; one SQLite I/O leg at a time, since the halves share the NVMe), a-p13 (untimed preservation,
`~/p13/p13run.sh`).

## ETAs given to the user (8 Oct)

- Column (3) with derived params: timing ~22:00 on 8 Oct; quotable after sqgen19f + A74 + R1-R5, ~9 Oct midday.
- Column (2), path B: C-COVER + multi-lock + B spec + smoke + audit + timing, ~10-11 Oct.

## Standing decisions of 7-8 Oct

- PM is widened and adopted (configuration fixed after init; no SQLite object outlives sqlite3_shutdown).
- A premise is replaced by a run-time check wherever that is cheap (P-SUBARRAY → carve; P-LOCK-STABLE checked;
  P-TEMP-OWN by write inventory).
- Measure time, not check mass: zgBp's three params were ~0.1 % of checks and +4.8 % of time.
- The memcached figure to quote is 1.80× over upstream. 1.96× over same-root stock counts the 9c656 regression and is
  not quoted.
- FFmpeg preservation rests on seeded races (8/8 kept). Redis's seeded path was re-verified on a-p13.
- The harness voids runs whose client did not do full work, on all apps (HARNESS-FIXES 7-9).

## PARKED 8 Oct 16:50 (all lanes confirmed)

Each lane's state, with branch heads and its next step:
- tsan-dev: sqgen-rules.md "## tsan-dev state (… parked …)".
- tsan-dev-2: evconf-checked.md §15 (two archives), and sqgen-rules.md "Multi-lock pass".
- tsan-dev-3: sqgen-rules.md "PARKED tsan-dev-3 (8 Oct 2026, 16:4x)". sqgen19g is NOT frozen: the discriminating
  fixture (xls) is open, then freeze and a delta audit.
- tsan-exp: experiments.md §2, "Unattended operation and readout".

Running unattended (no session needed):
- apollo A: zgf4, path B's field-only ceiling (zqs | zg00 | zgF0 | zgBe), until ~17:10. Then two detached focs chains:
  - zrc4, the root check (zqs | e52d stock | zgB | zgBe | A/A), ~17:15-19:00;
  - zcf4, column (3) on tsan-carve-ad1ae6e3aa43 (stock | zgBe2 | zg19 | zg19f | A/A, TSAN_CARVE_STATS=1),
    ~19:05-22:15.
  Logs: $C/zrc_chain.out, $C/zcf_chain.out, $C/cov1chain.log; apollo ~/p5-apollo/cov1/.
- focs: tsan-carve-2f6f3143fcc6 is frozen (PASS 16:26:49).

Path B status at parking:
- The static rule (lcB/brule.py) keeps a field only if C-COVER keeps it, lockcand finds 0 unheld sites, and it is not
  an owner field.
- specB/sqlite3-B2.spec (40c13b2601a31b8f) keeps 17 of 38: BtShared 4 and IntegrityCk 13 (a stack object, so
  irrelevant). The rule drops pSchema and nRef by itself. Smoke: 3/3 clean.
- With 4 BtShared fields, B's column (2) is likely ≈ 1.0×. Read zgf4 (whose ceiling covers 34 fields) and the -stats
  elision count before timing B2. Expect to propose stopping B.

Decisions pending with the user:
- premise P-SUBOBJ (sub-object bounds; an extension of IN-BOUNDS) and "debug-location fidelity" (the carve narrowing),
  both from A73/A74; recommended: adopt;
- path B's future, after zgf4 and the B2 count.

Cleanup for later: tsan-dev-3's leftover scratch directories, listed in its parked section.

## RESUMED 8 Oct 18:15 in account claude-focs2

All five sessions were copied and renamed; lanes resumed from their parked paragraphs. zgf4 ended ~16:55 (read by
tsan-exp after the resume), zrc4 runs since 16:56, zcf4 follows by the detached chain. Memory and scratchpads are
under ~/.claude-focs2 now.

## 9 Oct 00:50: night plan (user: everything on SQLite, all servers)

- apollo B: pad sensitivity item, then the LO-MARK leg at N=6 (measurement), then kmg6 offsets 00/16.
- apollo A: blocked by another user since 23:34 (rule (e) retires cells); a quiet-watcher starts the LO-MARK record
  leg and then a re-run of zdc4 when it frees.
- focs (Intel): timed legs announced in /extra/alexey/FOCS_LEG: F1 LO-MARK (zvS | zvG | zvL | A/A), F2 column (3)
  (stock | gen9b | gen19j | gen19j-roots | A/A). Dev lanes build and run untimed work on a-p13 meanwhile.
- Dev: name-free lock in LO-MARK (multi-lock pass + lockcfg3; tsan-dev-2, tsan-dev-3), LO-MARK v2 for BtCursor and
  pInfo (design in sqgen-rules.md, audit A80 running; tsan-dev pass prep, tsan-dev-3 generator).
- Open with the user: six LO-MARK premises (list in audit-a79-lo-mark-final-delta.md, Q6 (ii)).
- memcached (2): two holes in the owner-marker runtime fixed (audits A70k-A70m), root of record
  tsan-ecc-b54ce0196884, PRESMC PASS; kmg6 half B 1.776 (offset 48 at 1.737: placement), other half pending.

## State 9 Oct 10:30 (after the night on SQLite)

Column (2), LO-MARK (lock markers in shadow, no assertions, no annotations, lock found by the configuration filter):
- v1 record leg zvl4 (quiet AMD half, root tsan-lomark-718c7520052c): 1.260x over stock, gen9b 1.233x on the same root; 0 voids. Other legs: 1.140 (noisier half), 1.058 and 1.075 (Intel). v1 and gen9b are not separated.
- v2 (cursors; audits A80, A81; preservation PASS; lmrepro7 69 ok): first leg on Intel level with v1 (1.086 against 1.075) while eliding 6.4 times more.
- Defect found 9 Oct: always-on shared atomic counters on the marker paths (about 750 ns per access under contention). Fixed on root tsan-lomark-913cae99c263 (counters per thread; gate PASS). Legs z9 (stock | gen9b | v1 | v2 | A/A) run on focs until ~13:12 and queue on the quiet AMD half.
- The per-thread memo of the ordering verdict was rejected by audit A83 (a thread in flight across a reset); it stays a branch.
- Waiting for the user: the six premises of v1 (audit-a79-lo-mark-final-delta.md, Q6 (ii)). Until then column (2) is entered as a measurement.

Column (3), premise B2 replaced by run-time checks:
- sqgen19n: gen19j's directives + four owner checks (page producers, cursor page in Insert and Delete); B2 (ii) gone for the six cell params. sqgen19o: + two read-path field checks, B2 (ii) gone for the pInfo params too (31-37 M more tests per run).
- Root tsan-carve-da51c2dd8b3c, gate PASS; audits A82 and A82 Delta (SOUND WITH CONDITIONS; D1 closed by the per-load gate).
- Leg zh (stock | gen19j | gen19n | gen19o | A/A) queued on the AMD half after z9 and on focs after z9. It prices the checks; the tables keep "rests on B2" until it reads.
- A stale dominator tree after carve's block split was found and fixed; the final IR of the timed gen19j arm is identical with and without the fix.

memcached column (2): 1.80x over upstream on the repaired runtime (leg kmg6); the under-repair mark is removed.

Open: placement sensitivity (unresolved; needs 10+ runs per pad on the quiet half); the two-archive runtime plan (postponed); the other user's load on apollo since 09:44.
