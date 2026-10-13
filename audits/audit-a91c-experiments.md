# Audit A91c: experiment state and schedule for the camera-ready

14 Oct 2026, about 01:00. Read-only: files were read, nothing was built or run, no host was contacted.

Sources read: `measurement-protocol.md` (whole), `paper-results.md` (whole, as of 14 Oct 00:24),
`optimization-results.md` (sections 1, 2 and greps), `experiments.md` (section 2 head, rows 1121-1153, the
sections of 12-13 Oct from line 4138 to the end, sections 8.8, 8.10, 8.12 and greps), `tools/progress.py` and its
registry, `audits/audit-a89-mapping-vs-arms.md`, and in the paper repository `4-experiments.tex`, `4-plots-*.tex`,
`appendix/chromium-selected.tex`, `appendix/contention.tex`, `appendix/rex-details.tex`,
`plan/camera-ready-plan.md` and the diff of the branch `redis-results`.

Short names: PR = `paper-results.md`, OR = `optimization-results.md`, EXP = `experiments.md`, PROTO =
`measurement-protocol.md`. "PR:37" is line 37. "Removal" = DE in its removal form (dominance and post-dominance);
"verified" = DE with `-tsan-de-verified`.

## 0. Verdict

1. **Speed is nearly done; the untimed metrics are not.** After the SQLite and FFmpeg record cells of this night,
   AllOpt, slowdown over native and memory are final under the adopted lines for all five applications. What
   silently went stale with the switch of DE's form is everything counted or built from the binaries of
   SQLite, FFmpeg, Redis and memcached: static counts (M4), runtime entries (M5), code size (M7), compile time
   (M6), and the two thread-count sweeps. None of them has a removal-form figure on file. All but the sweeps
   are untimed or take seconds per build, and fit into two hours of the Intel host.
2. **A native Chromium build is on record.** EXP:2400 records a native build with the camera-ready compiler
   (binary hash `d2de71d51a32ad2e`, built 9 Oct on the non-timed host, copied to the Intel host as
   `chrome-orig-8345` in the Chromium out directory). PR:127 and PR:275 say it is still to be made. It has never
   been run. If the directory still exists, Chromium's slowdown over native and memory need no build, only one
   Speedometer leg of under four hours. Whether it still exists was not checked (outside this audit's scope).
3. **The thread-count sweep exists only in the old form of DE** (memcached, FFmpeg) and does not exist for
   SQLite's gain at all (PR:455-460). A SQLite series cannot be produced by a run: no workload with a thread
   parameter reaches the marked objects.
4. **The tracker holds no queue for 13-14 Oct.** The newest plan in EXP section 2 is of 9 Oct, and the registry
   read by `tools/progress.py` was last changed on 30 Sep. The queue described in the task exists only in
   sessions. Section 4 below is written to be pasted there.
5. Everything essential ends by about 09:30 on 14 Oct, before the meeting. The night of 14-15 Oct is spare.

## 1. Matrix: application x metric

States: FINAL; OLD-FORM = final, but measured with verified DE where removal is now adopted; CURRENT;
PROVISIONAL; MISSING; NOT-NEEDED. "tonight" = the record cells reported as ending 00:45 (FFmpeg) and running to
03:15 (SQLite); neither is in the files yet, so their cells are judged by what the files hold.

### 1.1 Speedup over stock (M1)

| cell | SQLite | FFmpeg | Redis | MySQL | memcached |
|---|---|---|---|---|---|
| STC | FINAL, = stock (PR:37) | FINAL (PR:38) | FINAL (PR:39) | FINAL (PR:40) | FINAL (PR:41) |
| SWMR | FINAL, = stock | FINAL | FINAL | FINAL | FINAL |
| LO | FINAL 1.26 | FINAL | FINAL, = stock | FINAL | FINAL |
| EA | FINAL 0.97 | FINAL | FINAL | FINAL | FINAL 1.86 (PR:41, 64-66) |
| DE | OLD-FORM 1.01 (PR:37) | OLD-FORM 1.02 (PR:38, 68) | FINAL, removal 1.11 (PR:39) | FINAL, verified = adopted | OLD-FORM 0.98 (PR:41 says so) |
| DE+Peeling | OLD-FORM 1.00 | OLD-FORM 1.01 | FINAL, removal 1.08 | FINAL | OLD-FORM 1.00 |
| DynSTC | FINAL 1.00, not the adopted line's (PR:74-76) | FINAL 1.28 (PR:67) | FINAL 1.14 from a noisy leg (PR:64, A/A 0.985) | FINAL, not the adopted line's | FINAL, not the adopted line's |
| FE-INL | FINAL | FINAL | FINAL | FINAL 1.09 | FINAL |
| AllOpt | OLD-FORM 1.23 (PR:37); removal screening 1.27 is CURRENT (PR:247-249); record tonight | OLD-FORM 1.27 (PR:38: "DE in its verified form"); record tonight | FINAL 1.47 (PR:39); quiet re-read optional | FINAL 1.10 (PR:40) | FINAL 1.91 (PR:41) |

The DE form does not touch the STC, SWMR, LO, EA, FE-INL and DynSTC arms: by their flag lines (EXP:4138-4216,
A89 section 2) none of them contains DE, so their binaries are the same under either decision. Only DE,
DE+Peeling and AllOpt need a removal re-read; that is six bars (three applications x two) beyond tonight's cells.

Chromium (no DE in any adopted configuration, so the switch does not apply): the four analyses 1.005, + DE
1.012, + peeling 0.993, CR1 1.104 / 1.101, AllOpt 1.286 / 1.248: FINAL (PR:98-104), without intervals.
Speedometer 3.0: CR1 1.112 FINAL; AllOpt 1.225 PROVISIONAL (N = 2, PR:113). Single-configuration bars per
analysis as in the submitted figures: MISSING and not obtainable in time (see section 2).

### 1.2 The other metrics

| metric | SQLite | FFmpeg | Redis | MySQL | memcached | Chromium |
|---|---|---|---|---|---|---|
| M2 slowdown over native | OLD-FORM 5.43 / 4.40 (PR:131); tonight | OLD-FORM 2.76 / 2.17 (PR:136); tonight | FINAL 6.21 / 4.22 (PR:132) | FINAL 10.03 / 9.11 (PR:134) | FINAL 8.48 / 4.44 (PR:133) | MISSING (PR:127); native build on record, never run |
| M3 memory | OLD-FORM -1.7 % (PR:131); tonight | OLD-FORM -0.3 % (PR:136); tonight | FINAL -0.5 % | FINAL -0.7 % | FINAL 10 % | CURRENT, over stock only: 1.06x / 1.54x, an increase (PR:138-139); the paper's quantity needs the native run |
| M4 static counts, binary | OLD-FORM, marked current (PR:150) | OLD-FORM (PR:153-154) | OLD-FORM (PR:152) | CURRENT (PR:155), should read final: verified is adopted | OLD-FORM, lower bound (PR:151) | MISSING (EXP:4354-4365) |
| M4 by the pass's statistics | OLD-FORM, marked final (PR:170) | OLD-FORM (PR:171) | OLD-FORM (PR:172) | FINAL (PR:173) | OLD-FORM (PR:174) | MISSING |
| M5 dynamic | CURRENT, OLD-FORM, runtime entries only (PR:200-201) | CURRENT, OLD-FORM, and with both options (PR:199) | CURRENT, OLD-FORM (PR:190-192) | CURRENT (PR:255-258) | CURRENT, OLD-FORM (PR:189) | NOT-NEEDED (PROTO:90) |
| M6 compile time over native | CURRENT, across windows, OLD-FORM (PR:273) | OLD-FORM, marked final (PR:271) | CURRENT, across windows, OLD-FORM (PR:272) | CURRENT, means of 2 builds (PR:274) | OLD-FORM, marked final (PR:270) | MISSING over native (PR:275); +10 % over stock FINAL (PR:288) |
| M7 code size | OLD-FORM 1.18x (PR:150) | OLD-FORM 1.40x (PR:153) | OLD-FORM 2.58x (PR:152) | CURRENT 1.31x (PR:155), final in fact | OLD-FORM 1.06x (PR:151) | CURRENT 3.0x, main binary only (PR:259) |
| M8 preservation, adopted line | CURRENT, removal, 10 runs (PR:364-367) | CURRENT, removal, 9 seeds x 10 (PR:360-362) | CURRENT, removal, 11 seeds x 10 (PR:369-373) | FINAL in substance, 10 against 10 (PR:399); marked nowhere | CURRENT, removal, 5 runs per arm, protocol asks 10 (PR:351-354) | CURRENT (PR:312-320); one rate unexplained, section 3 |
| M8 single bars | FINAL (PR:322-325); DE bars removal CURRENT; DynSTC bar MISSING | CURRENT (PR:375-385) | CURRENT (PR:327-344) | CURRENT, 7 runs (PR:387-397); FE-INL and DynSTC bars MISSING | FINAL (PR:346-349); DE bars removal at 5 runs; DynSTC bar MISSING | NOT-NEEDED |
| M9 suite | FINAL both forms (PR:428-438) | FINAL both forms; 12 failures, none a lost report | FINAL both forms | FINAL | FINAL both forms | FINAL (PR:404) |
| thread-count sweep | FINAL but empty: 1.00 on a workload the line does not reach (PR:455-460); the gain has no series | OLD-FORM (PR:464-469) | NOT-NEEDED | NOT-NEEDED | OLD-FORM at 8, 6, 4; PROVISIONAL at 2 (PR:447-450) | NOT-NEEDED |

### 1.3 What the removal form changes, per quantity

| quantity | changes with the form? | what suffices |
|---|---|---|
| AllOpt speedup, slowdown over native | within noise where screened (removal over verified 1.00, 1.00, 1.007, 1.001; PR:39, 248; OR section 1), but the paper must quote the adopted binary | the record cells (done or running) |
| DE, DE+Peeling bars | yes: Redis's DE moved from 1.06 to 1.11 (EXP:1149) | one singles-size leg per application |
| memory | no visible change (memcached 100 MiB in both, Redis 965-970) | taken from the record cell, no extra run |
| static counts, Q and Q_call | **yes, materially.** Under verified DE a covered access keeps an inline test, so Q does not fall (PR:157-162); under removal the check is deleted, so Q falls. This is the figure that decides how the paper's static-reduction plot looks | untimed recount on the existing removal trees (disassembly, about 2 min) plus one compile per line with the pass's statistics saved |
| runtime entries per operation | little for entries into the runtime; the paper's L (checks executed in any form) was never measured in either form | untimed recount (0.25 s trace windows) |
| code size | yes: fewer inline tests, added `_keep` entries (EXP:4917 ff.) | `size` on the existing trees, untimed |
| compile time | may change in either direction; not measured | three builds per arm in one quiet window on the Intel host; builds take 5-90 s |
| preservation | must be shown per form | done for all five (memcached at 5 runs) |
| regression suite | yes, and it is on file: SQLite, memcached and MySQL rows are clean under removal (PR:433-434) | done |
| sweeps | the 8-thread memcached point reads 1.90 verified against 1.91 removal; FFmpeg 1.001 | re-run with the adopted binary, or one sentence saying which form was measured |

## 2. Figures and tables of the paper's evaluation

| figure or table | data points | final under the adopted configurations? | missing |
|---|---|---|---|
| Slowdown over native (`fig:tsan_slowdown_native`), 6 applications x 2 bars | 12 | 6 of 12 (Redis, MySQL, memcached); 4 more after tonight's cells | Chromium, both bars. Caption says Speedometer 3.1; the tree has 3.0 (PR:21) |
| Speedup panels, memcached, Redis, FFmpeg, SQLite (`fig:all_results`), 9 bars each | 36 | 28 | DE and DE+Peeling of memcached, SQLite, FFmpeg (6); AllOpt of SQLite and FFmpeg (2, tonight) |
| MySQL panels (`fig:results_mysql_select`, `_write`) | 2 x 9 | the data are final, but for another workload: the write set of three scripts (PR:40), no `select-random-points` panel | a decision on the panels (one composite panel, or per script); per-script figures of the single arms other than FE-INL are not in PR |
| Chromium panels in the main text (Parser, Image Decoder) and appendix (Speedometer, Paint, SVG), 8 bars each | 40 | no | per-analysis bars do not exist on the camera-ready compiler; only the five cumulative configurations do (PR:98-104). Needs the new design the plan already names, not a run |
| Chromium distribution by suite (`fig:chromium_aggregated`) and the CDF in the appendix | per-story speedups | data exist (49 + 48 stories, N = 3); not in PR | per-suite quartiles and an interval, computed from files on hand |
| Static reduction (`fig:sir_reduction`), 6 bars | 6 | 1 (MySQL) | removal recount for 4; Chromium has no count; and the open decision on what is plotted (PR:157) |
| Dynamic reduction (`fig:dir_reduction`), 5 bars | 5 | 0 as defined; "drop in runtime entries" exists for all five, current, old form for four | removal recount; the paper's definition has no data in either form |
| Unique locations (`fig:dlir_reduction`) | 5 | not measured again (PROTO:26-27) | the subfigure and its paragraph go, or keep submitted data with a statement |
| Memory overhead reduction (`fig:mem_overhead`), 6 bars | 6 | 3; 5 after tonight | Chromium (native run). Four of five applications show an increase, against the text's claim |
| Compile time over native (`fig:cto_overhead`), 6 bars | 6 | 0 strictly: memcached and FFmpeg are old form, Redis and SQLite also across windows, MySQL a mean of 2, Chromium absent | one window for four applications; a third MySQL round; Chromium over native |
| Contention (`fig:results_runtime_contention`), memcached, SQLite, FFmpeg | series | none under the adopted lines | memcached and FFmpeg in removal form, both without the single configuration that PROTO:79-81 asks for; SQLite has nothing to show |
| Race detection paragraph, preservation protocol | all configurations shown | adopted lines yes (two at "current"); bars: DynSTC of three applications and MySQL's FE-INL missing | see 1.2 |
| Oracle and ReX tables | | not re-measured by decision | nothing |

## 3. Inconsistencies inside `paper-results.md`

Stale or self-contradicting text:

1. PR:3 "State: 12 Oct" over content of 14 Oct; PR:4-5 defines two statuses, the file uses three.
2. PR:33 "All cells but AllOpt wait for the final record legs" against "final" in every row below it.
3. PR:126-127 "Other applications: pending" against a table that holds all five.
4. PR:64 "not timed yet" and "REPLACED ... reads 1.14" in one sentence.
5. PR:73 "until it is in, the table keeps 1.24" against 1.27 in PR:38; PR:49-50 "the adopted line over upstream
   stock: pending" against 1.26 in PR:38.
6. PR:82-83 "preservation of Redis's single configurations: pending" against PR:327-332.
7. PR:84-85 describes FFmpeg's DynSTC column as the option on the stock binary (copy 1.79, h264 0.93); the cell
   in PR:38 is the replaced arm, 1.28.
8. PR:45-48 (memcached base): "AllOpt is 1.89 and 1.81", "about 8.05x slower than native", "a leg that separates
   compiler from runtime is queued". The row says 1.91 and 1.82, PR:133 says 8.09, and the leg ran (EXP:1134:
   the runtime).
9. PR:399-400 "Per-site rates: pending" although the per-site paragraphs above it exist.
10. PR:282 "Redis ... current (static placement under test)" against PR:285 "not adopted".
11. PR:263 heads section 5 "Over the stock TSan build"; the first table is over native.
12. PR:151 gives memcached's guarded checks as a lower bound (>= 313); PR:174 has the exact 527.

Two legs quoted for one thing:

13. FFmpeg per transcode: PR:38 (copy 2.53, h264 1.01; cell without `phase_guard`) against PR:90-92 (2.44 and
    0.93; "the final leg", which had both options). PR:199 (runtime entries) is also "both options".
14. FFmpeg compile time: PR:93 "+18 % against about +100 %" matches neither PR:271 (+38 % and +108 % over native)
    nor PR:281-283 (+17 % and +86 % over stock).
15. Redis AllOpt appears as 1.47 (PR:39), 1.52 (PR:54), 1.49 (PR:282) and 1.51 (PR:285); over upstream stock as
    1.43 (PR:39) and 1.47 (PR:54). The base difference itself reads 2.1 % in one leg and 3.6 % in another.
16. Speedups repeated in section 5 are the old ones: SQLite 1.23, FFmpeg 1.29 and 1.24, MySQL 1.13, memcached
    1.81, Redis 1.49 (PR:280-287).
17. MySQL compile time from two legs: 925.0 / 1075.9 s (PR:274) and 874.75 / 1015.13 s (PR:286); both +16 %.
    PR:274 says "Release", PR:19 says `-O2 -g` for the timed trees: the build type of the compile-time trees and
    of the timed trees differ, or one label is wrong.
18. "Checks in stock" has two denominators: the linked binary (PR:150-155: 57,997; 6,748; 37,941; 514,609;
    599,000) and the pass's statistics (PR:170-174: 58,778; 6,911; 38,557; 522,110; 993,181). MySQL's differ by
    66 %; PR:164-166 explains duplication by peeling but not this gap in stock. The "share of stock" column
    depends on which is used.
19. FFmpeg's suite: 391 tests run (PR:411-413) against 392 (PR:425-426).

Base:

20. PR:58-59: every figure is over the root's own stock, the choice "to be decided". OR section 1 quotes
    memcached over upstream stock (1.82) and the other four over the root's stock. The roots differ from upstream
    by 0 % (MySQL), 1.2-1.9 % (FFmpeg), 2-4 % (Redis), 4.5-4.7 % (memcached), unresolved (SQLite). Under an
    upstream base the figures are 1.26, 1.43, 1.10, 1.82 and about 1.25 (SQLite, not resolved).

AllOpt below one of its bars:

21. SQLite: LO 1.26 [1.22, 1.31] above AllOpt 1.23 [1.20, 1.27] (PR:37). Different leg sizes; the same trees read
    1.28 with the other half idle. In a bar chart LO will stand above AllOpt.
22. FFmpeg: DynSTC 1.28 [1.26, 1.30] above AllOpt 1.27 [1.27, 1.28] (PR:38). In the one leg that holds both, 1.280
    against 1.271 (OR section 1).
23. Chromium on the AMD host: CR1 1.057 above AllOpt 1.017 on the hold-out sample, and Speedometer 1.082 above
    1.006 (PR:106, 112-113). The paper takes the Intel figures; the AMD ones contradict them and are marked final.

Intervals that reach 1.0, shown as bars:

24. SQLite: every column but LO is inside the singles leg's A/A, 1.014 [0.938, 1.073] (PR:80-81); that A/A also
    exceeds the validity bound of PROTO:61 (2 % as a mean) only in its interval, per offset not checked here.
    FFmpeg DE 1.02 [1.00, 1.03], DE+Peeling 1.01 [1.00, 1.02]; Redis EA 1.01 [1.00, 1.02]; MySQL EA 1.01
    [1.00, 1.01]. Chromium: + DE 1.012 against a noise floor of 1.013 (PR:101, 106); Speedometer AllOpt on AMD
    1.006 inside the A/A range. By PROTO:60 these are "no measurable change".
25. memcached DE 0.98 [0.96, 1.00] comes from one of two offsets (PR:86).

Statuses that contradict the text:

26. "final" with verified DE where removal is adopted: SQLite and FFmpeg rows (PR:37-38), SQLite and FFmpeg in
    PR:131, 135-136, the statistics table (PR:170-172, 174), compile time (PR:270-271, 280-281, 287), both sweeps
    (PR:447-449, 466-469).
27. "current" where nothing further is planned and the form is the adopted one: MySQL's static row (PR:155),
    the removal-form preservation paragraphs (PR:354-373).
28. PR:460 marks the SQLite sweep "final"; as a statement about the gain it is absent.
29. Chromium rows are "final" without an interval; PROTO:86-88 asks for a bootstrap over stories. The camera-ready
    plan's requirement of run counts and variance throughout applies.

Not a bookkeeping matter:

30. PR:318: the PartitionAlloc race is reported by stock in 11 of 21 runs and by AllOpt in 2 of 19. No site is
    lost by the rule, but the rate falls fivefold and OR section 2 calls it "not explained". PR:316: crashpad
    33 of 33 against 10 of 19, attributed to eviction although stock never misses. Both belong in the
    preservation text as they are.
31. PR:138-139 and four of five rows in PR:131-136: memory rises. The paper's sentence "we observe a reduction
    in memory overhead for almost all programs" has no support in the new data.
32. PR:219-227: SQLite's figure counts statements of which the data-modifying ones fall. The metric's name is
    an open decision; no run changes it.

## 4. Schedule, 14 Oct 01:00 to 15 Oct 09:00

Durations are those of the same leg shapes on 13 Oct (EXP:1124-1153): SQLite record 2 h 12 to 2 h 45, SQLite
4-arm screening 34 min; memcached record 2 h 54 to 3 h 54, 4-arm screening 53 min, sweep about 3 h; FFmpeg record
1 h 53 to 2 h 15, 6-arm screening 36 min, sweep 1 h 58; Redis record cell 1 h 11; Chromium Speedometer on the
Intel host about 24 min per sanitized run (15 runs in 6 h 03, EXP:2541).

Constraints used: nothing compiles on a host during a timed leg there; at most one SQLite or MySQL leg on the AMD
host at a time (EXP:1296); the whole Intel host is quiet during a Chromium leg; a figure may not fall below OR
section 1 (SQLite 1.23, FFmpeg 1.27, Redis 1.47, MySQL 1.10, memcached 1.82 over upstream, Chromium 1.25).

### AMD host, half A (SQLite, memcached)

| # | start-end | leg | unblocks | priority |
|---|---|---|---|---|
| A0 | running to 03:15 | SQLite record cell, removal | AllOpt bar, slowdown, memory, upstream base for SQLite | 1 |
| A1 | 03:20-04:05 | SQLite DE and DE+Peeling, removal (stock, `s9q`, `s9u`, A/A; 2 offsets x N = 2) | 2 bars | 1 |
| A2 | 04:10-05:10 | memcached DE and DE+Peeling, removal (stock, `m4q`, `m4u`, A/A) | 2 bars | 1 |
| A3 | 05:15-08:15 | memcached sweep with the removal line at 6, 4, 2 server threads (8 is the record cell): stock, `kcv`, and the EA arm `m4V` as the configuration that carries the gain; N = 2 at every point | the memcached contention panel; a clean peak figure in place of the flagged 2.23 | 2 |
| A4 | 08:20-09:35 | idle on purpose (B3) | | |
| A5 | 09:40-10:50 | SQLite, CPU-count series of PR:229-237 with the removal line, bare command, 3 alternating runs per variant | a substitute for the SQLite contention panel, if the meeting wants one | drop first |

### AMD host, half B (FFmpeg, Redis, MySQL)

| # | start-end | leg | unblocks | priority |
|---|---|---|---|---|
| B0 | ended about 00:45 | FFmpeg record cell, removal | AllOpt bar, slowdown, memory | done; to be read and entered |
| | 00:45-04:05 | idle on purpose | SQLite's record and singles read with the other half quiet: 1.276 quiet against 1.233 busy on the same trees (OR section 1), and 1.23 is the floor. Half B has about 4 h of work for 32 h, so the wait costs nothing | |
| B1 | 04:10-04:50 | FFmpeg DE and DE+Peeling, removal, static placement (stock, `f2q`, `f2u`, A/A; the every-check twins only if that line stays in the paper) | 2 bars | 1 |
| B2 | 04:55-06:25 | FFmpeg sweep with the removal line and `dynstc_rt=1` only at 16, 8, 2 threads (4 is the record cell): stock, `fkr`, DynSTC arm | the FFmpeg contention panel | 3 |
| B3 | 08:20-09:35 | Redis record cell repeated with half A idle and no copies to the disk | no new paper item; may lift 1.47 toward the morning's 1.52 | 4, drop second |

No MySQL leg is needed: its adopted line keeps the verified form and every MySQL cell stands.

### Intel host (untimed work first, then Chromium)

| # | start-end | work | unblocks | priority |
|---|---|---|---|---|
| F1 | 01:00-01:20 | static recount and `size` on the removal trees (adopted lines and DE, DE+Peeling arms of SQLite, FFmpeg, Redis, memcached); disassembly only, 8 jobs | M4 and M7 for four applications: static-reduction plot, code size | 1 |
| F2 | 01:20-01:50 | one compile of each removal line with the pass's statistics saved | the table of PR:168-174 in removal form, memcached's exact guarded count | 1 |
| F3 | 01:50-02:30 | compile-time window: native, stock, adopted removal line alternating, 3 builds each, memcached, Redis, SQLite, FFmpeg; nothing else on the host | 4 compile-time bars, and the "across windows" mark of Redis and SQLite goes | 1 |
| F4 | 02:30-03:10 | runtime entries per operation on the removal lines (trace windows, counts only); FFmpeg with `dynstc_rt=1` only | dynamic-reduction figures for four applications in the adopted form | 2 |
| F5 | 03:10-03:30 | check that `chrome-orig-8345` exists with hash `d2de71d51a32ad2e`; one untimed Speedometer start | decides F6 | 1 |
| F6 | 03:30-07:15 | Chromium Speedometer leg: native, stock, CR1, CR1+N1, N = 3, order rotated; announced as timed, no compile on the host | Chromium's two slowdown bars, its memory bar, and N = 3 or more for AllOpt on Speedometer | 2 |
| F7 | 07:20-08:20 | Chromium compile time over native: CPU time on the 400 sampled units, native, stock, CR1+N1 | the sixth compile-time bar | 3 |
| F8 | 08:20-09:15 | MySQL third round of builds (native, stock, adopted) | median of 3 in place of a mean of 2 | drop third |
| F9 | any gap after | Chromium linked-site count, stock against CR1+N1 | a Chromium bar in the static plot | drop first |

Compile-time measurements can only go on the Intel host, in a window with no Chromium leg: the AMD host times
legs all night, the third host is shared and not for timing. Hence F3 before the Chromium leg, F7 and F8 after.

### Non-timed host

| # | work | unblocks | priority |
|---|---|---|---|
| P1 | memcached, removal: extend 5 to 10 runs per arm (stock, `kcv`, `m4q`, `m4u`) | preservation at the protocol's N | 1 |
| P2 | DynSTC bars of SQLite, memcached, MySQL (the run-time option on the stock binary) and MySQL's FE-INL arm, 10 against 10 | the four bars shown without a preservation result | 2 |
| P3 | Chromium Speedometer with reporting on, 10 stock and 10 CR1+N1 | the PartitionAlloc rate (2 of 19 against 11 of 21) | drop first |
| P4 | only if F5 fails: native Chromium build (8888 s at 36 jobs last time), then copy | F6 and F7 move to about 08:00-13:00 and still precede the meeting | insurance |

### Rank, paper items per machine-hour

F1 + F2 (0.8 h: static plot and code size for four applications) > F3 (0.7 h: four bars) > A1, B1, A2 (under
1 h each: two bars each) > F6 (3.75 h: three bars and a run count) > F4 > F7 > A3 (3 h: one panel) > B2 > B3 >
A5, F8, F9, P3.

### Whether the two named items fit

- **Native Chromium build and a run with it:** the build need not be made if EXP:2400's copy survives; the run
  (F6) fits tonight. If the copy is gone, the rebuild on the non-timed host and the run still fit before the
  meeting.
- **Thread-count sweeps under the adopted lines:** they do not exist. memcached and FFmpeg need the runs A3 and
  B2 (or one sentence naming the form measured: the forms agree within 1 % at the record points). SQLite's cannot
  be measured; A5 is a CPU-count series, a different statement.

### Cut-off

- A record-size leg (up to 4 h) starts no later than 14 Oct 22:00.
- A singles-size leg (up to 1.5 h) or a Chromium repetition starts no later than 15 Oct 03:00.
- Nothing starts after 15 Oct 04:30: a result needs reading, the ledger cross-check and an entry in PR before
  09:00.

The afternoon and night of 14 Oct stay free for what the meeting decides (base, SQLite's metric and panel, how
static reduction is shown) and for the repeat of any leg whose A/A fails.

### Deliberately left out

The paper's own compiled DynSTC arm; a dominance-only DE variant (the verified-against-removal ratios stand in
for the ablation); rebuilding the EA and SWMR arms without MEMINTR and SWMR-ROOTS; unique-location counts;
oracle and ReX; per-analysis Chromium bars; Chromium on the AMD host; the Blink sample on the native build; a
SQLite thread sweep; MySQL in removal form; the paper's L by a counting root.

## 5. Not checked

- The live state of any host or leg. The FFmpeg removal cell (reported ended 00:45) and the running SQLite
  cell are not in the files as of 00:27; their figures and validity are unknown here.
- Whether `chrome-orig-8345` still exists, and whether the harness runs a native build.
- Whether the removal single-arm trees are already on the AMD host.
- Raw per-run files: no figure was recomputed from them. The arithmetic of PR:129-136 (slowdown ratios, memory
  reductions) was recomputed from the table's own numbers and agrees.
- Per-offset A/A values against PROTO:61.
- That the non-DE single arms are unchanged by the form: taken from the flag lines on file, not from binaries.
- The F and P durations without a precedent of the same shape (F1, F2, F4, F7, P1-P3) are estimates.
