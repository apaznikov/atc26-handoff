# tsan-improve handoff, 28 Oct 2026 (account move)

Read this first when resuming. Russian translation: tsan-improve-handoff-2026-10-28.ru.md.

## Role and lanes

- **tsan-improve** (this lane, tmux `2-tsan-improve`, config `~/.claude-focs2`) coordinates the measurements of the
  upstream PR versions, the PR texts, and what is left of ATC'26.
- **tsan-exp** (tmux `2-tsan-exp`) runs every build, leg and readout. Its queue runs as unattended chained scripts on
  focs, plus race-preservation runs on a-p13. Registration and amendments: `/extra/alexey/upr-2026-10-20/REGISTERED.txt`;
  readouts: `/extra/alexey/upr-2026-10-20/READOUT-*.txt`.
- **tsan-rt** (tmux `2-tsan-rt`, cloned from this lane on 26 Oct) owns the runtime-optimization track for a second
  paper (apollo). Not this lane's work; coordinate only on shared hosts and upstream overlap.

## The results file

`/home/alexey/atc26-handoff/upstream-pr-results.md`, published with
`env -u GITHUB_TOKEN FILE=upstream-pr-results.md MSG=... BODY=... bash publish-results.sh` from `/home/alexey/atc26-handoff`
→ https://github.com/apaznikov/atc26-handoff/blob/main/upstream-pr-results.md

Conventions (all decided by the user, 26-27 Oct):
- every number is a percentage against stock, (stock time / arm time − 1) × 100;
- verdict rule, stated in the legend: 95 % interval against A/A from the same leg; 🟢 lo > A/A hi, 🔴 hi < A/A lo,
  "=" point inside A/A, 🟡 otherwise; † A/A wider than ±3 %;
- in the DE + runtime fix column, 🟰 also means "no race report in 5 census runs, so the changed runtime code never runs";
- no single-threaded programs and no 1-thread runs (a race detector is not run on them);
- every cell re-derived from the readouts by an audit on 26 Oct; keep doing that for new cells.

## State of the measurements (28 Oct ~15:00)

Done and in the table:
- DE + runtime fix: timing of SQLite, memcached, MySQL at -O0/-O2 (all "=" except SQLite tests, see footnote ¹);
  race preservation for SQLite, memcached, MySQL and FFmpeg (HEVC slice-threaded decoding, added 27 Oct): no site lost
  in any arm; Redis has no racy workload (io-threads 20/8/4 tried).
- Census (a-p13): zstd, xz, brotli, LevelDB, OpenSSL (-O0/-O2), 7-Zip, FFmpeg -O0/-O2, Redis -O0 report no race → 🟰.
- Kernels at 8 threads re-timed (I11): only NPB LU W resolves (DE +103 %, fix +101 %, DE + EA +116 %); the rest stay
  🟡 or "=" because the kernels' own run-to-run spread on focs is 8-20 % (no TSan reset seen; binding, passive wait,
  larger input and more runs do not help); water_nsquared gets no leg.
- SQLite -O0 re-timed with all arms: single tests 🟢 in every arm (checkpoint_starvation_1 +67-77 %).
- Redis -O2 LRANGE: DE and DE + fix 2-3 % slower (footnote ⁶); profile queued (I12-5).
- Chromium 156 (stock vs DE + EA): unresolved; blink_perf.dom modify-element-classname ~3 % slower in 3/3 reps.

Running or queued on focs (tsan-exp chains, in order), expected end ~31 Oct-1 Nov:
1. I11 apps: memcached -O0 EA, the small 🟡 cells (OpenSSL, xz, brotli, 7-Zip, pigz), RocksDB -O0/-O2 at N 8.
2. NPB LU class A redo (i11kA2: e, g, r; it failed on 28 Oct through a script bug).
3. I12: DuckDB r build; Chromium tsan-upde and tsan-upea builds, r relinks; I12-2 legs s/a/d/r for pigz, DuckDB,
   hotspot, ocean_ncp, streamcluster, NPB BT A; cri9n2 (Chromium, REPS 5, dom REPS 8, arms s/a/d/g/r, + e if its code
   differs; pre-stated modify-element-classname check); I12-5 Redis LRANGE perf profile.
4. AMD (apollo) only after all Intel work, if time allows (user, 26 Oct).

On resume: read every READOUT newer than this file and tsan-exp's log `/extra/alexey/upr-2026-10-20/STATUS-LOG.md`,
fold the numbers into the table (re-derive verdicts by the rule), publish, report to the user.

## Upstream PRs

- Decided 26 Oct: the runtime fix goes into #169897 itself (one PR). Local branch `upstream-next/dom-final-rt`
  (4a134ba07eae) in `/extra/alexey/upstream-next/repo`: NFC hoist → runtime fix (4139d1043f04, audited, no blocker) →
  DE + test `dominance_cover_report.c`. check-tsan 383 / 0 failed, TSan IR lit 26 / 0, negative control shows the test
  catches the gap. Not pushed.
- Draft texts: `/extra/alexey/upstream-next/pr-texts/169897-description.md` (runtime paragraph added),
  `rt-keep-description.md` (superseded as a separate PR, kept for notes). Old figures in the PR descriptions must be
  replaced by the table's numbers once the measurements end.
- Push rules: nothing to the LLVM PR branches without the user's explicit yes per push; no AI trailers.
- Next steps after the measurements: write the final PR description with the numbers, show it to the user, then the
  reviewers' asks (melver for #169897, vitalybuka on EA's form).

## ATC'26

- TAPS: the third proof (26 Oct 19:07) has all 48 corrections plus the introduction comma; PDF and HTML checked.
  The user has to press Approve in the TAPS dashboard (after opening both versions).
- Open for the user: presenter form (was due 23 Oct), reply to the chairs, shepherd comment, A7 answer, CHECK PAPER
  DETAILS, invitation letter if needed, hotel G-ACM1, Concur/Trip.Biz travel request, talk (14 + 4 min, 16:9) and A0
  poster. Conference 15-18 Nov, Hong Kong.

## Pending decisions for the user

1. AMD measurements: when, if at all.
2. Whether the unresolved kernels (lavaMD, srad, particlefilter, water_spatial) are reported as 🟡 with their consistent
   point estimates, or dropped from the PR text.
3. When to push `upstream-next/dom-final-rt` to #169897 (after the final numbers and the description are approved).
