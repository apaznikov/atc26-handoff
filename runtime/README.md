# TSan runtime track (tsan-rt): plan

State 26 Oct 2026. On 26 Oct Alexey opened this track as a second paper: optimizations of ThreadSanitizer's runtime
(compiler-rt/lib/tsan) that keep every race stock TSan reports. It is run by the lane tsan-rt (a fork of tsan-improve);
tsan-improve keeps the upstream PRs and the ATC'26 work.

**Files here:**
- `ideas.md`: the inventory of every runtime-side idea (status, best measured effect, soundness class, code), the
  ranked shortlist, the gaps no idea addresses yet (G1-G11) and the contradictions found between files (K1-K18).
- `als.md`, `dd-exact-ab.md`, `n1-atomic-a081.md`, `range-hit-skip.md`, `sid-alias.md`, `slot-pref.md`: the detailed
  notes of single ideas and censuses (all are in this folder; the earlier remark that they are kept elsewhere was wrong).
- `code/`: branch tables of the ATC tree (`branch-newrt.tsv`, `branches-all.tsv`, `notinR.tsv`).

## Setup

- **Repository.** focs-lab/llvm-project-tsan (public), branch folder `tsan-rt/`:
  - `tsan-rt/base` = upstream main 00a2b7021c23 (19 Oct 2026), the same base as the upstream PR branches;
  - one idea per branch, `tsan-rt/<idea>`, as one logical commit with its tests on top of base;
  - `tsan-rt/measure` merges the ideas a leg needs and is never proposed upstream.
  - Local clone: /extra/alexey/tsan-rt/repo. Remote `paper` = the ATC tree; `paper/camera-ready` = 8345a0396fa5, whose
    runtime is 34 files, +4430 lines over upstream of 6 May 2024. Ideas built there are ported, not merged.
- **Hosts.**
  - apollo (AMD EPYC 9115, 2 sockets x 16 cores): node0 (CPUs 0-15,32-47) for this track's timed legs; node1 is wtc's,
    which measures continuously; stretto stops its untimed work on node0 during our legs. Every leg opens and closes a
    START/END line in apollo:~/APOLLO_LEG.log and is announced to wtc and stretto (agreed 26 Oct).
  - Cross-socket noise is measured, not assumed away: each run records the busy share outside its CPU set, every leg has
    an A/A, and a verdict at the edge gets a re-check in a quiet window agreed with wtc.
  - Builds on a-p13 (Docker image tsan-p13-build, clang-18, nice/ionice). All arms of one leg come from the same builder.
- **Gates for every idea, in order.**
  1. A design note with the soundness argument: exact against stock, or the named premise.
  2. check-tsan, plus new tests with a negative control (the test must fail without the idea's safeguard).
  3. An independent audit (Opus, read-only).
  4. Race preservation on the applications (N = 10, rotated CPU sets, verdict per race site).
  5. Timing: A/A, interleaved arms, 2 code layouts, 95 % bootstrap intervals, registered before the leg; verdict rule as
     in ../upstream-pr-results.md.

## Phases

| # | phase | content | estimate |
|---|---|---|---|
| P0 | Infrastructure | build tsan-rt/base on a-p13; trees of SQLite, FFmpeg, memcached, Redis, MySQL (Release) for apollo; port the leg harness to node0 with the APOLLO_LEG protocol; A/A calibration of node0 while wtc runs on node1 | ~2 days |
| P1 | Where the runtime spends its time | perf profiles of stock TSan on every application, attributed to runtime components (access fast path, CheckRaces miss path, ranges, mutex path: slot lock / SyncVar / clocks / trace, deadlock detector, allocator and StackDepot, interceptors, function entry/exit, resets), plus counters (resets, preemptions, evictions). Settles K5-K7 in ideas.md and sets the targets; it is also the paper's motivation figure | ~2 days |
| P2 | Ideas already built, ported to upstream | ALS (R6, audited exact; Chromium +2 %, never run on the servers); DD-EXACT A + B1 (R8, R9; memcached +19-22 %, lock-order reports differ, see decisions); RT-SYNC as the negative result that points at the slot lock rather than the clock join | ~1 week |
| P3 | New designs, in the order P1 ranks them | mutex path S1/S3/S2b and a SyncVar cache (R2-R4, R7; gap G1); vector-clock representation (G2, e.g. tree clocks); allocator stack-id cache and freed fill (R17); empty-epoch release or lazy increment plus early reset (R19-R21); incremental DoReset (G3); cold shadow on ranges (G4); TSan's own lock contention (G5); unaligned fast path (G7) | weeks |
| P4 | Detection | wider epochs / more sids (R24), eviction victim policy (R23): the runtime then misses fewer races than stock. Needs a race-loss benchmark: reproducers plus per-site counts on the applications | weeks |

## Decisions wanted from Alexey

1. DD-EXACT B1 changes lock-order (deadlock) reports, not race reports: it reports some cycles earlier and can miss
   some stock reports after a graph flush (audit A34 H2). Accept as a separate class, or require a fix of the
   unchecked paths first.
2. Venue and date of the paper (outline below).

## Paper outline (draft, not yet approved)

- Working title: where ThreadSanitizer's runtime spends its time, and how to cut it without losing a race.
- Claims, each with what it needs:
  1. A component breakdown of the runtime's cost on real servers (P1).
  2. Exact optimizations of the synchronization path, the dominant cost on lock- and atomic-heavy programs (P2, P3).
  3. Optimizations of fresh-memory and range handling (P3).
  4. A runtime that loses fewer races than stock under bounded shadow state (P4).
  Every claim: speed-up on the applications at 8 threads with A/A, race preservation, check-tsan, and an upstream form.
- Venue: to be chosen once P1 shows the size of the effects; the 2027 cycles of ASPLOS and EuroSys have closed.

## Earlier decisions (ATC period)

- 25 Sep: the runtime ideas were parked, since the ATC paper was about static analysis and instrumentation.
- 28 Sep: everything about the runtime moved into this folder; censuses only.
- 29 Sep - 10 Oct: RT-SYNC was built, gated and timed null on all five applications; RT-SYNC-LF built, one unresolved
  leg; ALS built, audited (A84) and timed on Chromium; DD-EXACT built and timed; all then parked (10 Oct: the runtime
  is not to be touched during the camera-ready).
- 26 Oct: the track reopened as a second paper (this file).
