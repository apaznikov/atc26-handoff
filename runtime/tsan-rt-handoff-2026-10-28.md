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
| `tsan-rt/als` | d9b877c7e046 + 24c4088f486b | ALS ported to upstream, plus the audit fixes. Squash into one commit before any PR; the auto-mode classifier blocked force pushes, so it needs Alexey's OK |
| `meta-shadow/base`, `meta-shadow/{parallel-note,shadow-resident-filter,workload-profiler}` | 73e729f7b2f0; 3995d5da1c07, c883c4524e3a, 8c9c383c792b | Ownership Meta-Shadow Memory project, imported from its fork |
| `l0-filter/base`, `l0-filter/main` | 4d87e55d8a41, d7a513c92648 | L0 filter project, imported |
| `aba/base`, `aba/main` | 8947e494f92e, 12582630f5d4 | ABA detection project, imported; another lane will own it |

- **Push recipe:** `env -u GITHUB_TOKEN GIT_CONFIG_GLOBAL=/home/alexey/.agent-creds/gitconfig-focs git push origin ...`. The tsan-paper token gets 403 on this repo.
- **Local clone:** /extra/alexey/tsan-rt/repo.
  - Remotes: `paper` = the ATC tree /home/alexey/dev/llvm-project-focs-lab (`paper/camera-ready` = 8345a0396fa5, `paper/als`), and the three project forks.
  - Worktree: /extra/alexey/tsan-rt/wt/als.

## Pending decisions (Alexey)

1. **Project branches.** The imported `*/main` (and the meta-shadow) branches already hold the work, so PRs into them would be empty. Two options:
   - reset them to their bases: a force push, blocked by the classifier until he says "сбросить";
   - or have PRs target new branches `<folder>/dev`.

   The texts for the three contributors (PR instructions) are in projects/contributor-messages.md; send them after this decision.
2. **DD-EXACT B1** (memcached +19-22 %): race reports are unchanged, but deadlock (lock-order) reports differ (audit A34 H2). Accept as a separate class, or fix first.
3. **Paper outline and venue:** README.md, "Paper outline". Venue to be chosen after P1; the 2027 cycles of ASPLOS and EuroSys are closed.
4. **Where the L0-filter contributor may run benchmarks:** focs is booked for timed legs until about 29 Oct. Options: we measure their branch with our protocol, or a focs window later.

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

## Next steps, in order

1. Finish P0 (STATE.md lists what is left). Then the A/A calibration leg on node0, announced first.
2. P1: perf profiles of stock TSan on the five applications, by runtime component.
3. ALS timing legs (memcached, Redis, MySQL, SQLite, FFmpeg), with race preservation.
4. After the decisions above: the DD-EXACT port, the project branches, the outline.

## Rules that bite in this lane

- No AI trailers, English in repos, no personal names or roles in committed text (contributors are named by project).
- Never touch other lanes' directories on a-p13 (~/pres-i10, ~/rtkeep-*, ~/p13) or on apollo (~/p5-apollo, ~/upn-all, wtc/stretto dirs).
- Background waits: do not leave `until ... sleep` loops running. On 28 Oct four such loops of the P0 subagent had been waiting for a day after their dry run had ended; they were stopped.
