# tsan-rt lane: handoff, 28 Oct 2026

Read this first when the lane resumes (new account or new session). Plan and inventory: README.md and ideas.md in
this folder. Russian translation: tsan-rt-handoff-2026-10-28.ru.md.

## Lane

- **tsan-rt** = the TSan runtime-optimization track for a second paper.
  - It was created on 26 Oct as a fork of tsan-improve: tmux session `2-tsan-rt`, `claude --resume <tsan-improve session> --fork-session -n tsan-rt`, CLAUDE_CONFIG_DIR=~/.claude-focs2.
  - tsan-improve keeps the upstream PRs (#169897 with the runtime fix folded in, #169896, #192945), the focs legs I10/I11, Chromium and the ATC'26 camera-ready. Message it only for coordination.
- **Mandate** (Alexey, 26 Oct): runtime ideas on apollo, a second paper, the repo focs-lab/llvm-project-tsan with its own branch folder, a thorough audit and an experiment plan. Implementation is allowed in this lane. "Делай всё в публичном": the branches are public.

## Repository: github.com/focs-lab/llvm-project-tsan (public)

| branch | commit | what |
|---|---|---|
| `tsan-rt/base` | 00a2b7021c23 | upstream main of 19 Oct; base of the track (same as the upstream PR branches) |
| `tsan-rt/als` | a87606ee8322 | ALS ported to upstream with the audit fixes, squashed into one commit on 28 Oct (Alexey's OK); same tree as 24c4088f486b, which a-p13's root-als was built from |
| `meta-shadow/base`, `meta-shadow/{parallel-note,shadow-resident-filter,workload-profiler}` | 73e729f7b2f0; 3995d5da1c07, c883c4524e3a, 8c9c383c792b | Ownership Meta-Shadow Memory project, imported from its fork |
| `l0-filter/base`, `l0-filter/main` | 4d87e55d8a41, d7a513c92648 | L0 filter project, imported |
| `aba/base`, `aba/main` | 8947e494f92e, 12582630f5d4 | ABA detection project, imported; another lane will own it |

- **Push recipe:** `env -u GITHUB_TOKEN GIT_CONFIG_GLOBAL=/home/alexey/.agent-creds/gitconfig-focs git push origin ...`. The tsan-paper token gets 403 on this repo.
- **Local clone:** /extra/alexey/tsan-rt/repo.
  - Remotes: `paper` = the ATC tree /home/alexey/dev/llvm-project-focs-lab (`paper/camera-ready` = 8345a0396fa5, `paper/als`), and the three project forks.
  - Worktree: /extra/alexey/tsan-rt/wt/als.

## Decisions of 28 Oct (Alexey)

1. **Project branches stay as imported.** Alexey tells the contributors that everything is ready. They continue in their forks and send new changes as PRs into their folder's branches. The drafts in projects/contributor-messages.md are not needed.
2. **DD-EXACT: taken into work.** Next step: port `experiment/dd-exact-ab` (944fb9f4c502: options `deadlock_detector_large_table`, `deadlock_detector_exact_prelock`) from the ATC tree to `tsan-rt/dd-exact`, then the gates and timing.
   - My reading of "берем в работу": the lock-order difference (A34 H2) is accepted for measuring, and it is stated as its own class: race reports exact, deadlock reports may differ.
   - Re-measure on the record workload, where the detector was only 2 % of memcached.
3. **Paper outline and venue:** later.
4. **The L0-filter contributor measures on his own PC.**
5. **Squashing tsan-rt/als:** done, force push allowed for this branch.

## State of the work

- **Inventory and plan:** runtime/ideas.md has 26 runtime-only rows (R), 24 compiler + runtime rows (C), 5 project rows (P), the shortlist, gaps G1-G11 and contradictions K1-K18. runtime/README.md has the setup, the gates and phases P0-P4. Both are published in apaznikov/atc26-handoff.
- **Project audits:** runtime/projects/meta-shadow.md and l0-filter.md.
  - The shadow-resident filter is exact but has the same mechanism as N1; the new parts are portability and slot-0-first.
  - The parallel note is a negative result with 2 holes.
  - The L0 filter is unsound as built.
- **ALS:** ported, audited twice (A84 on the ATC tree; a port audit with no blocker, fixes applied in 24c4088f486b).
  - On a-p13: check-tsan 380 passed / 0 failed / 93 unsupported / 1 expected failure. atomic_load_skip.c passes in the static and dynamic configs; all three mutants bite.
  - poll.c: 2.04 M of 2.43 M loads skipped, verify clean.
  - Deferred for an upstream form: strip the census and mutants; relaxed atomics on TidSlot::thr; lazy or smaller cache.
- **a-p13** (builds; Docker image tsan-p13-build, ≤24 jobs, nice/ionice):
  - roots ~/tsan-rt/root-base (stock) and ~/tsan-rt/root-als;
  - build dirs ~/tsan-rt/build-{base,als};
  - scripts ~/tsan-rt/{run.sh,build.sh,als-tests.sh} (`V=<variant> COMMIT=... ./run.sh <steps>`);
  - logs ~/tsan-rt/logs/<variant>/.
- **apollo:**
  - node0 (0-15,32-47) is ours for timed legs. node1 is wtc's: it measures continuously, and the noise is recorded, not gated.
  - stretto vacates node0 during our legs.
  - Every leg writes a START/END line in ~/APOLLO_LEG.log and is announced to wtc and stretto. No leg of ours has run yet.
  - App trees and the harness are in ~/tsan-rt/ (P0); status in apollo:~/tsan-rt/STATE.md (see below).

## P0 status (apollo/a-p13)

See apollo:~/tsan-rt/STATE.md, written at the stop of 28 Oct.

- **Ready to run.** No timed leg has started. Nothing of this lane is running on apollo or a-p13.
- **Trees.** apollo:~/tsan-rt/apps/{base,als}/{sqlite,ffmpeg,memcached,redis,mysql}, built on a-p13 in tsan-focs-sb:24.04 with upstream-next's apps.sh (-g -O2; MySQL RelWithDebInfo as in the upa legs).
  - Layout-offset trees: `.o32`; SQLite also `.o16` and `.o48`. Each offset tree pads the runtime with uncalled bytes and relinks; this was checked.
  - Provenance: TREES.txt, PROVENANCE.txt, IDENTITY.txt. Base and als differ in code only through path strings; checked object by object.
  - a-p13 hit its 20 GB floor, so the MySQL and FFmpeg trees there were removed after verifying the apollo copies. A MySQL relink needs a ~9 min rebuild.
- **Harness:** apollo:~/tsan-rt/harness/.
  - `leg.sh [--dry-run|--smoke] <app> <tag> "<offsets>" <N> <arms>`, with arms as `name=tree[:TSAN_OPTIONS]`, e.g. `s=base a=base f=als n=als:atomic_load_skip=1`.
  - Workload on 4-15,36-47, harness on 0-3,32-35, interleaved arms, A/A.
  - Records the busy share of the bench set, the harness set and node1, the frequency, and foreign time on node0. It refuses to start if a container is pinned onto the bench set.
  - `readout.sh` = focs's rr_final.py (READOUT-upa0* format, 95 % bootstrap).
- **Dry run** (one untimed run each, wall s): base / als off / als on.
  - memcached 373 / 355 / 356; Redis 241 / 240 / 181; SQLite 242 / 237 / 225; FFmpeg 140 / 130 / 129; MySQL 358 / 342 / 343.
  - Leg estimate for 33 cells: memcached and MySQL ~3.3 h each, SQLite 2.2 h, Redis 2 h, FFmpeg 1.2 h.
- **ALS census.**
  - memcached: 0 atomic loads (a null case).
  - Redis: 70.3 G acquire loads with a sync object, 99.9 % hits. That looks like spin-waiting, so a Redis gain would partly measure faster spinning; say so.
  - MySQL: hits 66-98 % per workload (write_only 2.45 G loads, 98 %).
- **Before the first leg:**
  - Register the protocol (arms, N, offsets, retirement rules, decision rule).
  - Announce to wtc and stretto. stretto's containers ran on 4-15,36-47 on 27 Oct 02:20-04:20, which is allowed between legs.
  - Frequency is not fixed (schedutil, cell means 2.9-3.9 GHz). It is recorded per cell; the A/A arm carries it.
- **Rule slips:** two short probes ran on apollo unpinned (under 10 s).

## Results of 29-30 Oct (timed legs on apollo node0; details in apollo:~/tsan-rt/RESULTS.md)

- **Branches added:** `tsan-rt/als` = e078bcda770c (release fences instead of seq_cst, audited SOUND WITH NOTES);
  `tsan-rt/als-lazy` = 8e71159fffdf (versions only on sync objects a load has cached; audited SOUND WITH NOTES, notes
  applied; measured as 9a22221f0aca, hot paths identical but for the global read); `tsan-rt/dd-exact` = d30b65678d04.
- **Layout confound found 30 Oct:** the `.o<k>` offsets moved the program but not the runtime's own code, so each
  runtime build was timed in one internal layout. Runtime-internal layout alone swings MySQL by +/- 3 %. All legs from
  30 Oct use `.i00/.i16/.i32/.i48` trees (pad first in libclang_rt.tsan.a, second pad keeps main's class); built for
  base, als, alsr, alsl, ddx on all five apps.
- **ALS, final under iNN (speed-up over stock, 95 % CI; verdict vs A/A):**
  - Redis: l 1.211 [1.156, 1.262] GREEN, r 1.198 GREEN (99.9 % of hits are spin-wait loads; say so).
  - memcached: two replicates, side by side: l 0.998 / 1.004 "=" (flag cost removed), r 0.996 "=" / 0.998 YELLOW.
  - MySQL (after a 7-cell top-up): A/A 1.001, r 0.996, l 0.989, all "=".
  - FFmpeg: r 1.004, l 1.003, "=".
  - SQLite (re-measure p1is, quiet, 0 retired; verdict on STEADY, registered M12 before the leg: walthread1/2,
    checkpoint_starvation_1/2, stress2): r 0.997, l 0.992, "=". dynamic_triggers and stress1 swing +/- 40-60 %.
- **DD-EXACT (both options), final under iNN:** memcached D 1.030 [1.014, 1.042] GREEN (+31 MiB RSS); MySQL 0.998,
  Redis 1.010, FFmpeg 0.999, SQLite STEADY 0.991, all "=". The earlier +19-22 % on memcached does not reproduce on the
  record workload, and the single-layout +1-3 % on MySQL and FFmpeg were runtime layout.
- **SQLite frequency** reads 2.36-2.40 GHz in every leg: threadtest3 keeps ~2 of 14 bench CPUs busy, and the monitor
  averages over all 14. A property of the metric, not other load.
- **Race preservation (gate 4), 30-31 Oct, untimed; no site LOST anywhere** (apollo:~/tsan-rt/RESULTS.md, p2/):
  - memcached, SQLite: every stock site KEPT by l and D; l is vacuous there (no skippable acquire loads).
  - Redis, FFmpeg: stock reports no race on their workloads (Redis probes with unit tests and mixed io-thread load
    also 0), so seeded trees were used (ATC seeds, one compiler, three runtimes, .text identical): every seed kept in
    10/10 runs in s, l, D (Redis S5, a signal-timing seed: s 24/30, l 28/30, p 0.25).
  - ALS-aimed Redis seed A4 (reader caches the version before the writer's release, 100 rounds): clean in 20/20 for s
    and l, a false race in 3/3 under the no-bump-join mutant, so the check can fail.
  - MySQL at N=20: rare-site UNDETERMINED rows (stock 1/20) in l and D; a stock-vs-stock arm (M18) gives as many, so
    they are sampling noise.
  - check-tsan with the options forced into every test: alsl 862 and D 870 statuses identical to plain, 0 FAIL.
- **P1-MUTEX** (apollo:~/tsan-rt/p1/P1-MUTEX.txt, DWARF call graphs): memcached's pthread mutex path is 36.3 % of
  server samples; slot-mutex CAS 12.7 % and SyncVar s->mtx CAS 13.1 % of the whole profile; clocks, trace, lookup < 2 %.
  Three SlotLock and three s->mtx pairs per lock+unlock (from source). Rough S3 ceiling ~17 % of memcached CPU.
- **P1 profiles:** apollo:~/tsan-rt/p1/P1-SUMMARY.txt (SlotLock 51.6 % of Redis, mutex path 29 % of memcached,
  fast path 70 % of SQLite).
- **Open question to Alexey:** start the design of merging SlotLock across the mutex path (S3)?

## Next steps, in order

1. Register the protocol, announce to wtc and stretto, then the A/A calibration leg on node0.
2. Port DD-EXACT to `tsan-rt/dd-exact` (build and gates on a-p13; the tree recipe is in STATE.md), alongside step 1.
3. P1: perf profiles of stock TSan on the five applications, by runtime component.
4. Timing legs for ALS and DD-EXACT on the five applications, with race preservation.
5. The paper outline, when Alexey returns to it.

## Rules that bite in this lane

- No AI trailers, English in repos, no personal names or roles in committed text (contributors are named by project).
- Never touch other lanes' directories on a-p13 (~/pres-i10, ~/rtkeep-*, ~/p13) or on apollo (~/p5-apollo, ~/upn-all, wtc/stretto dirs).
- Background waits: do not leave `until ... sleep` loops running. On 28 Oct four such loops of the P0 subagent had been waiting for a day after their dry run had ended; they were stopped.
