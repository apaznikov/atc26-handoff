# Audit A92: metrics the camera-ready needs that are neither measured nor queued

14 Oct 2026, about 05:30. Read-only: files were read, nothing was built or run, no host was contacted.

Object: the paper at `origin/app-results` (c7da925; stacks on `redis-results`; `methodology` and `fe-inl` are NOT
ancestors of it), read with `git show`. Compared against `paper-results.md` (PR, last change 14 Oct 05:07),
`optimization-results.md` (OR), `experiments.md` (EXP; "Queue of the night 13-14 Oct and of 14 Oct" and the three
sections after it), `measurement-protocol.md` (PROTO, M1-M9), audits A91a-c and the plan on `main`.

Taken as given (not re-proposed): the running or queued items named in the request (memcached and FFmpeg thread
sweeps, MySQL both-forms record cell and DE singles, "Sound" arms of memcached, SQLite, Redis, FFmpeg, Redis quiet
re-read, Chromium Speedometer 3.0 with native arm and the hold-out native leg, Chromium compile time over 400
units, compile-time reduction candidates, preservation runs on the third host). "queued" below means one of these.

States: IN = entered on the branch; FINAL = value in PR with status final; CUR = status current; QUEUED;
MISSING = not measured and not queued; DROP = dropped or not re-measured by an earlier decision.

## 1. Inventory: every figure, table and numeric claim

### 1.1 Main text, evaluation

| item (file:line on `origin/app-results`) | shows | data per application / bar / point | in the branch? |
|---|---|---|---|
| `4-experiments.tex:11-17` methodology | hosts, N, configurations | hosts and N: text exists on `methodology` only; Chromium N = 3 without offsets, Redis DE bars 2 x 4, sweeps 2 x 1 are not covered by it | no (other branch) |
| `:78` range of stock slowdown | "2.8x to over 19x" | low end FINAL 2.76; high end: Chromium over native QUEUED (else MySQL 10.0) | 2.8 yes; 19 no |
| `:78-80` stock over native per application | 5 numbers | SQLite 5.22, memcached 8.48, FFmpeg 2.76, Redis 6.21, MySQL 10.03: all FINAL | yes (5.2, 8.5, 2.8, 6.2, 10.0) |
| `:80`, `1-introduction.tex:45` Speedometer 3.65 -> 0.19, 19.2x | Chromium native and stock score | QUEUED (native arm, ends about 07:25) | no (6 numbers) |
| `4-plots-original-slowdown.tex:73,77` Fig. slowdown, Chromium pair | 19.2 / 16.8 | QUEUED | no |
| `:90-102` Fig. slowdown, 5 applications x 2 | stock / AllOpt over native | Redis 6.21 / 4.22, MySQL 10.03 / 9.11 (verified form; removal QUEUED), FFmpeg 2.76 / 2.17, memcached 8.48 / 4.44, SQLite 5.22 / 4.22: FINAL | yes, all 10 |
| `4-experiments.tex:91` memcached | AllOpt; DE, DE+Peeling "each 1.03x" | AllOpt 1.91 FINAL; DE 1.00, DE+Peeling 1.00 FINAL (removal) | 1.91 yes; 1.03 no |
| `:94` Redis | AllOpt 1.47, max single 1.14 | FINAL (quiet re-read QUEUED) | yes |
| `:97` FFmpeg | AllOpt; DE 1.30x; DE+Peeling 1.42x | AllOpt 1.27 FINAL; DE 1.01, DE+Peeling 1.01 FINAL (removal, static placement) | 1.27 yes; two no |
| `:100` MySQL | Select 1.16x; Write 1.10x; DE 1.15x / 1.11x | Select: MISSING (no select workload on record). Write set: AllOpt 1.10 FINAL in the verified form, removal QUEUED; DE 1.00 verified, removal QUEUED | 1.10 yes; three no |
| `:103-104` SQLite | AllOpt 1.24, best single 1.26 | FINAL | yes |
| `:107` Chromium | geomean 1.39x; Speedometer median 1.14x; Image Decoder 2.16x; Layout 1.45x | geomean over stories 1.248 (hold-out, N = 3) FINAL, 1.292 (first sample, N = 1) CUR; Speedometer Score ratio 1.225 at N = 2, repetition QUEUED; per-suite medians FINAL for the hold-out: layout 1.288, image_decoder 1.038 | no (4 numbers) |
| `4-plots-speedup.tex:45-116` Fig. speedup, memcached, Redis, FFmpeg, SQLite, 9 bars each | per-configuration speedup | all 36 FINAL | yes, all 36 |
| `:166-186` Fig. MySQL `select-random-points`, 8 bars | per-configuration | MISSING for every bar; PR has only runtime-entry counts for select scripts | no (old 8-bar set) |
| `:188-211` Fig. MySQL write, 9 bars | per-configuration | 7 bars FINAL; DE, DE+Peeling, AllOpt are the verified form, removal QUEUED | yes (verified values) |
| `:212-237` Fig. Chromium Parser and Image Decoder, 8 bars each | per single analysis on one story | MISSING: no single-analysis Chromium build exists on the camera-ready compiler; only five cumulative configurations (four analyses 1.005, + DE 1.012, + peeling 0.993, + FE-INL + MEMINTR 1.104, AllOpt 1.286; sample level, FINAL) | no (old 8-bar set) |
| `:241` `\Description` | 1.16x, 1.11x, 1.32x, 1.62x | write 1.10 FINAL; others as above | no |
| `:250-335` Fig. Chromium box plot per suite + caption (1.39x, 1.14x, 2.16x) | quartiles per suite, 7 boxes | per-story tables exist on the Intel host (48 + 49 stories); PR holds per-suite medians and whole-sample quartiles only; per-suite quartiles and whiskers: a recount, MISSING from PR; Speedometer box: only Score, no distribution | no |
| `4-experiments.tex:115-116` thread count | memcached and SQLite, "especially DE" | memcached, FFmpeg QUEUED (removal line + one single arm each); SQLite: no thread series can exist (PR 8) | no |
| `4-plots-instr-reduction.tex:50-55` SIR, 6 bars | (Q_orig - Q_opt) / Q_orig | by PROTO's definition, FINAL and negative in five of six: SQLite -24.8, memcached <= -1.3 (lower bound on guarded checks), Redis -8.6, FFmpeg -6.7, MySQL -11.0 (verified; removal -6.2 in EXP only), Chromium +0.6. Removed by the four analyses (pass statistics): 1.4 / 12.4 / 7.1 / 2.7 / 0.6 %, verified-form compiles for memcached, Redis, FFmpeg; removal-form statistics MISSING for those three; Chromium statistics MISSING | no |
| `:68` DIR, 5 bars | (L_orig - L_opt) / L_orig | L (checks executed in any form) MISSING in every application and not obtainable without counting code. Substitute "runtime entries": memcached 77.3, Redis 93.6, SQLite 9.9 (stress2), FFmpeg per transcode 85 / 18.6 / 5.0 / 2.2 with no composite, all FINAL; MySQL on its record workload MISSING (other sysbench scripts only, CUR) | no |
| `:80` DMIR, 5 bars; `4-experiments.tex:136-138` (43.9 %, 41.3 %) | unique locations | DROP (PROTO: not measured again) | no |
| `:94` OHR, 6 bars; `4-experiments.tex:148` (5.5, 6.0, 4.4 %) | memory overhead reduction | memcached 10, Redis -0.5, FFmpeg -0.3 (RSS from the verified-form cell), SQLite about -2, MySQL -0.7 (verified; removal QUEUED): FINAL. Chromium: QUEUED (needs native RSS); over stock 1.54x CUR | no |
| `4-experiments.tex:132, 134` | SIR 64.9 / 60.8 %, DIR 51.9 / 50.0 % | as the two rows above | no |
| `4-plots-overheads.tex:54` CTO, 6 bars; `4-experiments.tex:157-158` (2.5 %, 18 %) | (T_opt - T_nat) / T_nat | removal lines, one window, host not quiet, CUR: memcached +143, Redis +219, SQLite +74, FFmpeg +32. MySQL +25 CUR (mean of 2 builds, verified form). Chromium over native: QUEUED only if the 400-unit run includes a native arm (PR 5 says "pending (native build)") | no |
| `4-experiments.tex:162-166` race detection | no number in the text | adopted lines: no site lost (PR 6); several rows CUR; extensions QUEUED | n/a |
| `:182`, `:184` ReX | 95 %, 37 % | DROP (submitted data kept by decision) | unchanged |
| `:190` summary box | claims only | follows the bars | no |

### 1.2 Abstract, introduction, conclusion

| item | now | data | in the branch? |
|---|---|---|---|
| `abstract.tex:24` "up to 65 %" static | 65 % | see SIR: no positive figure for an adopted line except Chromium +0.6 %; the four analyses remove at most 12.4 % (memcached); DE alone 1.4-4.5 % (FINAL) | no |
| `abstract.tex:25` "23 and 58 %" dynamic | | runtime entries 9.9-93.6 % over four applications (FINAL), MySQL MISSING; a different quantity | no |
| `abstract.tex:25`, `1-introduction.tex:149`, `7-conclusion.tex:15` "up to 1.71x" | | 1.91x memcached FINAL (1.82x over upstream stock) | no |
| same three places, "Chromium 1.39x" | | 1.25x (hold-out) FINAL | no |
| same three places, "peaks 2.54x under high thread contention" | | memcached at 2 threads 2.23 CUR, re-run QUEUED; FFmpeg stream copy 2.52 FINAL (not a thread effect) | no |
| `abstract.tex:27` "1-18 %" compile time | | +25 % to +219 % CUR | no |
| `1-introduction.tex:45` | 3.65, 0.19, 19.2x | QUEUED | no |

### 1.3 Appendix

| item | shows | data | in the branch? |
|---|---|---|---|
| `4-plots-contention.tex:67-121` memcached, 8 series x 20 thread counts (2-40) | per configuration | QUEUED: stock, AllOpt, EA at 8 / 6 / 4 / 2. Other six series and counts above 8: MISSING | no |
| `:140-147` SQLite, 8 series x 20 | | no thread series possible; a CPU-count series of stress2 exists in the verified form (16 / 8 / 4 / 2 CPUs: 1.26 / 1.25 / 1.27 / 1.06, CUR); removal form MISSING | no |
| `:166-336` FFmpeg MJPEG, H.264, Passthrough, 8 series x 20 | per transcode | QUEUED: stock, AllOpt, DynSTC at 16 / 8 / 4 / 2; per-transcode series are in the legs' records (recount); other series MISSING | no |
| `appendix/contention.tex:2` | claims | follow the plot | no |
| `appendix/chromium-selected.tex:50-85` Speedometer, Paint, SVG, 8 bars each | per single analysis | MISSING (as the main-text Chromium panels); Speedometer has two configurations (1.112, 1.225) | no |
| `:112-139, 151` CDF; median 1.32x; about 93 % at or above 1.0 | per-story distribution | per-story tables exist; hold-out median 1.260, 47 of 48 at or above 1.00 FINAL; curve coordinates: recount | no |
| `appendix/rex-details.tex:7` oracle fractions; `:70-72` ReX table | | DROP (kept from the submission) | unchanged |

Not in the paper and measured: code size (M7, FINAL for all six), regression suite (M9, FINAL), per-site
preservation tables. They need no experiment; whether they appear is a text matter.

## 2. Gaps: needed, not measured, not queued

P1 blocks a main-text figure; P2 a main-text number; P3 appendix. Hours are estimates from the leg durations of
13 Oct (A91c, section 4), not measured for new shapes.

| # | gap | closes with | host, hours | priority |
|---|---|---|---|---|
| G1 | Chromium per-single-analysis panels (Parser, Image Decoder; appendix: Speedometer, Paint, SVG): 40 bars | cannot be closed by 16 Oct (eight builds and their legs on the one host that also times): needs a design decision. Minimal data route: one cumulative panel from the existing five configurations, per-story tables already on disk | recount, 0.5 h, no run | P1 |
| G2 | MySQL `select-random-points` panel and the two select numbers | wording or design decision (one MySQL panel over the write set). If the panel must stay: stock, FE-INL, AllOpt, A/A, 2 offsets x N = 2 on select_random_points | AMD B, 1.5 h | P1 |
| G3 | DIR panel: MySQL has no figure on its record workload; FFmpeg has no composite | MySQL: trace windows on the three write scripts, stock against the adopted line (and the removal line if it is adopted). FFmpeg: weight the four per-transcode figures by the record cell's function-entry counts (recount). The paper's L itself: cannot be closed, needs a wording decision (the panel becomes "runtime entries") | Intel, 0.7 h; recount | P1 |
| G4 | SIR panel: the plotted quantity is undecided; memcached's bar is a lower bound; no removal-form pass statistics for memcached, Redis, FFmpeg; MySQL removal row not in PR | one compile per line with statistics saved (the prepared 15 trees, 24 jobs; refused once as interference, so it needs an announced window). MySQL removal: copy the row already in EXP (Q +6.2 %, text 1.28x) into PR if removal is adopted | Intel, 0.5 h | P1 |
| G5 | DMIR panel and its paragraph | no experiment: dropped by decision; panel and three sentences need a wording decision | - | P1 |
| G6 | OHR panel: FFmpeg's figure uses the verified-form cell's RSS | recount from the removal record cell's own records | no run | P1 |
| G7 | CTO figure: every bar is CUR (window not quiet); MySQL is a mean of two builds in the verified form, "Release" against `-O2 -g` in PR 1; no MySQL removal-line build time | one quiet window after the compile-time reduction work is settled: native, stock, adopted alternating, 3 builds per arm, memcached, Redis, SQLite, FFmpeg (36 builds took 24 min on 14 Oct); MySQL one to three rounds of 48 min | Intel, 0.5 h + 0.8 to 2.4 h | P1 |
| G8 | Chromium box plot per suite | recount of per-suite quartiles and whiskers from the per-story tables (hold-out, N = 3). The Speedometer box has no distribution unless the runs kept per-test scores (not checked); else that box goes | no run, 0.5 h | P1 |
| G9 | Speedup bars carry no intervals; several PR cells are "1.00" with no interval (FFmpeg STC, SWMR, LO, EA, FE-INL; MySQL STC, SWMR, LO, DynSTC; memcached STC, LO, DynSTC, FE-INL) | recount from the singles legs' records; error bars are a plot change | no run, 0.5 h | P1 (run counts and variance are a camera-ready requirement) |
| G10 | SQLite's LO bar (1.26, 2 x 2) stands above AllOpt (1.24, 4 x 4); FFmpeg's DynSTC 1.28 above AllOpt 1.27 | SQLite: one leg with stock, LO, AllOpt, A/A at 4 offsets x N = 4. FFmpeg: the queued sweep's 4-thread point holds both arms in one leg; no new leg | AMD A, 2.75 h (disk I/O: not beside MySQL) | P2 |
| G11 | Chromium AllOpt on the first sample is N = 1 (1.292) | two more repetitions of stock and AllOpt on the first sample; not needed if the paper leads with the hold-out 1.25 | Intel, about 2-3 h (duration of one sample pass not checked) | P2, cut early |
| G12 | Abstract's "65 %" and "23-58 %" | follow G3, G4 and the open decision on how reduction is presented; no further run | - | P2 |
| G13 | Abstract's "1-18 %", `4-experiments.tex:157-158` | follow G7 | - | P2 |
| G14 | Base of the speedups: single-configuration bars exist over the root's own stock only | decision. Upstream base for the AllOpt bars exists in the record cells (memcached 1.82, Redis 1.43, FFmpeg 1.25, SQLite 1.23, MySQL equal); singles can be re-based only across legs | no run | P2 |
| G15 | If MySQL's removal line is adopted: its compile time, runtime entries and 10-run preservation do not exist (5 runs now); static row and suite row exist | compile time inside G7; entries inside G3; preservation 5 -> 10 runs for the adopted line, DE, DE+Peeling | third host, about 3 h | P2 |
| G16 | "Sound" row for MySQL and Chromium | none needed if stated: both adopted lines are decided at compile time (OR claim class 1), so Sound equals AllOpt | - | P2 |
| G17 | Chromium PartitionAlloc race: stock 11 of 21, AllOpt 2 of 19, unexplained (PR 6, CUR) | Speedometer with reporting on, 10 stock and 10 AllOpt | third host if it can run the browser (not checked), else Intel outside timed legs, 4-8 h | P2, cut early |
| G18 | Contention figure: six of eight series, thread counts above the pinned set, SQLite panel | cannot be closed: redesign to stock / AllOpt / carrier arm at four counts. Optional substitute for SQLite: the CPU-count series of stress2 with the removal line (5 variants, 3 alternating runs) | AMD A, 1.2 h | P3 |
| G19 | CDF over all Chromium tests | recount of the curve from the per-story tables | no run | P3 |
| G20 | Appendix Speedometer panel with the cumulative configurations (four analyses, + DE, + peeling have no Speedometer run) | 3 existing builds x 3 repetitions | Intel, about 3.6 h | P3, cut first |
| G21 | Oracle paragraph compares with (1 - DIR) of an AllOpt that no longer exists | wording only | - | P3 |

Closed by queued work, for completeness: Chromium slowdown pair, the three Speedometer scores, Chromium OHR and
CTO bars (if a native arm is in the 400-unit run: confirm), MySQL removal AllOpt / DE / DE+Peeling bars, the peak
figure (memcached at 2 threads), the Sound row of four applications, DynSTC and MySQL FE-INL preservation bars.

One rule to fix before the MySQL cell is read: "removal goes in if not below 1.10". The screening read removal
1.099 [1.092, 1.104] against verified 1.105; at two decimals that is 1.10, at three it is below. State the
comparison (rounded to two decimals, or the interval) before the result exists.

## 3. Final in `paper-results.md`, not yet on `origin/app-results` (numbers and plots only)

| file:line | old -> new | note |
|---|---|---|
| `4-plots-instr-reduction.tex:94` (OHR bars) | Memcached 0.5 -> 10; Redis 0.0 -> -0.5; FFmpeg 5.5 -> -0.3; SQLite 4.4 -> -1.9 (PR: "about -2 %", from 8 / 60 / 61 MiB); MySQL 6.0 -> -0.7 (verified cell; hold until the both-forms cell) | needs `ymin` at `:89` below zero and `ymax` above 10; Chromium bar stays stale |
| `4-experiments.tex:148` | 5.5 % -> -0.3 %; 6.0 % -> -0.7 %; 4.4 % -> about -2 % | values ready; the sentence reverses, so not enterable without wording |
| `4-experiments.tex:91` | DE and DE+Peeling "each 1.03x" -> 1.00x | inside a claim that no longer holds |
| `4-experiments.tex:97` | DE 1.30x -> 1.01x; DE+Peeling 1.42x -> 1.01x | same |
| `4-experiments.tex:107`; `4-plots-speedup.tex:331` | Chromium geomean 1.39x -> 1.25x (hold-out, N = 3) | "across all tests" becomes "over 48 stories" |
| `4-experiments.tex:107` | Layout median 1.45x -> 1.29x (hold-out; still the highest Blink suite there) | first sample: paint is highest (1.60, N = 1) |
| `4-experiments.tex:107`; `4-plots-speedup.tex:332` | Image Decoder 2.16x -> 1.04x (2 stories) | no longer the top of the range |
| `4-plots-speedup.tex:293-325` (box medians) | Parser 1.26 -> 1.04; DOM 1.25 -> 1.26; SVG 1.31 -> 1.27; Layout 1.45 -> 1.29; Paint 1.33 -> 1.21; ImgDec 2.16 -> 1.04 | quartiles and whiskers after G8; `ymax=2.8` can fall |
| `abstract.tex:25`; `1-introduction.tex:149`; `7-conclusion.tex:15` | 1.71x -> 1.91x (1.82x over upstream stock: base decision) | |
| `abstract.tex:25`; `1-introduction.tex:150`; `7-conclusion.tex:15` | Chromium 1.39x -> 1.25x | |
| `appendix/chromium-selected.tex:151` | median 1.32x -> 1.26x; "about 93 %" -> 98 % (47 of 48 stories) | with the recounted curve (G19) |
| `4-plots-speedup.tex:241` (`\Description`) | "1.16x and 1.11x" -> write 1.10x | alt text, not printed |
| `4-plots-original-slowdown.tex:84-105` | values already entered; the right-hand panel has no `\updfigstyle`, so the whole float still reads as stale | mark only |

Final but not enterable as a swap (metric undecided): SIR bars (`4-plots-instr-reduction.tex:50-55`), DIR bars
(`:68`), the abstract's 65 % and 23-58 %. CUR only, so not listed: CTO bars, 2.5 %, 18 %, 1-18 %.

## 4. Load plan, 14 Oct 07:00 to 15 Oct 03:00

Constraints applied: MySQL only on half B; SQLite and MySQL legs never overlap (one disk-I/O leg across the
halves); nothing compiles on the Intel host during a timed leg; builds at most 90 jobs; record-size legs start by
22:00, singles by 03:00. Times of running items are estimates.

### AMD host, half A (8 cores)

| start-end | leg | closes |
|---|---|---|
| running - about 11:15 | memcached sweep, first pass then second repetition (queued) | sweep panel, peak figure |
| 11:20-12:15 | memcached Sound arm (queued) | Sound row |
| 12:20-13:30 | memcached DE, DE+Peeling re-read without the declared co-load (2 x 2) | two bars read at threshold 0.40 instead of 0.10 |
| 13:35-14:15 | SQLite Sound arm (queued; after MySQL's legs end on B) | Sound row |
| 14:20-15:35 | SQLite CPU-count series, removal line (G18) | substitute for the SQLite contention panel |
| 15:40-16:55 | idle on purpose: Redis quiet re-read on B | |
| 17:00-19:45 | SQLite stock, LO, AllOpt, A/A at 4 x 4 (G10); no MySQL on B meanwhile | LO against AllOpt in one leg |
| 19:50-03:00 | reserve for the meeting's decisions and for a repeat of any leg whose A/A fails; filler: a third memcached sweep repetition | |

### AMD host, half B (16 cores)

| start-end | leg | closes |
|---|---|---|
| running - 11:40 | MySQL both-forms record cell (queued) | MySQL AllOpt, slowdown, memory in removal form |
| 11:45-13:15 | MySQL DE singles, removal (queued) | two bars |
| 13:20-14:00 | FFmpeg Sound arm (queued) | Sound row |
| 14:05-15:00 | Redis Sound arm (queued) | Sound row |
| 15:40-16:55 | Redis quiet re-read (queued; half A idle) | final status of 1.47 |
| 19:50-21:20 | only if the MySQL select panel is kept at the meeting: select leg (G2) | Fig. MySQL select |
| otherwise 19:50-22:00 | filler: second repetition of the FFmpeg sweep (per-transcode series at N = 2 x 2 twice) | tighter sweep intervals |
| 22:00-03:00 | reserve; nothing in this list needs it | |

### Intel host

| start-end | work | closes |
|---|---|---|
| running - 07:25 | Chromium Speedometer with native arm, hold-out native leg (queued) | slowdown pair, three scores, OHR bar, Speedometer N |
| 07:30-08:30 | Chromium compile time over 400 units, with a native arm (queued) | CTO bar |
| 08:30-09:00 | pass statistics in removal form, memcached, Redis, FFmpeg (G4); announced | SIR panel inputs, memcached's exact guarded count |
| 09:00-09:45 | MySQL runtime entries on the three write scripts (G3) | DIR panel's fifth bar |
| 09:45-16:00 | filler: compile-time reduction candidates, Redis first, then the whole-program step of memcached, then SQLite; recounts G1, G6, G8, G9, G19 beside it (no load) | lower CTO bars |
| 16:30-17:00 (+ 0.8 to 2.4 h) | final compile-time window, quiet, with the settled harness (G7); MySQL rounds after it | CTO figure, 2.5 %, 18 %, 1-18 % at final status |
| 19:30-22:30 | Chromium first sample, stock and AllOpt, two repetitions (G11); nothing compiles | first-sample figure at N = 3 |
| 22:30-02:10 | Speedometer on the three cumulative builds (G20) | appendix panel |

### Third host (untimed)

Queued preservation first (memcached removal to 10 runs, DynSTC bars, MySQL FE-INL bar, Sound arms). Then:
MySQL removal lines from 5 to 10 runs, started at 11:45 only if the record cell adopts removal (G15); then the
Chromium reporting runs (G17) if the host can run the browser.

### Cut order if time runs out

1. G20 Speedometer on cumulative builds. 2. G17 Chromium reporting runs. 3. Redis quiet re-read. 4. G18 SQLite
CPU-count series. 5. G11 first-sample repetitions. 6. FFmpeg sweep repetition. 7. G10 SQLite LO leg. Not to be
cut: G7 (final compile-time window), G3 (MySQL entries), G4 (pass statistics), the Sound arms, the recounts.

## 5. Inconsistencies between `paper-results.md` and the branch

| # | where | finding |
|---|---|---|
| 1 | Fig. slowdown legend and caption (`4-plots-original-slowdown.tex:79,112`): "best result achieved" | the bars are AllOpt. By the caption SQLite would be 4.1 (LO 1.26) and FFmpeg 2.1 (N1 at every check, 2.09); entered: 4.2 and 2.2 |
| 2 | `4-experiments.tex:103-104` | entered numbers contradict their sentence: "best individual reaches only 1.26x" beside AllOpt 1.24x |
| 3 | `4-experiments.tex:78-80` | entered numbers contradict their sentences: memcached 8.5 called moderate, Redis 6.2 called substantially higher |
| 4 | base | the branch says "vs. Original TSan"; every entered value is over the compiler root's own stock. Over upstream: memcached 1.82, Redis 1.43, FFmpeg 1.25, SQLite 1.23. OR's bold figure for memcached is the upstream one (1.82), for the other four the root's: the two files lead with different bases |
| 5 | MySQL workload | branch: "`Write-only`", sub-caption `write-only`; PR 1: composite of sysbench insert, update, delete (three scripts). PR 1 says `-O2 -g`, PR 5 and OR say Release |
| 6 | MySQL form | MySQL's entered bars and 9.1 are the verified form; PR 4 calls verified "MySQL's adopted form"; EXP's queue still lists the removal cell as dropped. All three precede the current decision |
| 7 | Redis and FFmpeg DE, DE+Peeling bars | PR: measured "with the inline test"; the branch's sentence at `:94` names the inline test as something only the full configuration adds |
| 8 | DynSTC bars of SQLite, memcached, MySQL; EA and SWMR bars of SQLite, FFmpeg, MySQL | PR 2: the DynSTC column is the run-time option, which those adopted lines do not use; EA and SWMR carry MEMINTR and SWMR-ROOTS. Nothing in the figure says so |
| 9 | "= stock" cells | Redis LO, SQLite STC and SWMR are plotted as 1.00; they were not timed (byte-identical to stock) |
| 10 | sign | PR 4 reports "Q over stock" (+24.8 % = more checks) and "runtime calls avoided" (negative = more calls); the paper's SIR is positive for a removal. A direct copy flips the meaning |
| 11 | denominators | "checks in stock" differs between the binary count and the pass statistics (SQLite 57,997 / 58,778; MySQL 599,000 / 993,181); a share of stock depends on which is used |
| 12 | DIR unit | paper: checks executed; PR: entries into the runtime, per request (memcached, Redis) or per 1,000 function entries (FFmpeg, SQLite), one 0.25 s window |
| 13 | compile time | PR 5 holds three tables with different bases and forms: FFmpeg reads +38 % (final, verified form, over native), +32 % (current, removal, over native), +17 % and +13 % (over stock). "final" there is the form no longer adopted |
| 14 | memory | FFmpeg's row is marked final with the RSS of the verified-form cell; SQLite's "about -2 %" is not a stated figure (8 / 60 / 61 MiB give -1.9 %) |
| 15 | Chromium geomean | two first-sample figures for AllOpt: 1.286 (sample level, N = 3, earlier harness) and 1.292 (per story, N = 1, corrected harness); the hold-out reads 1.248 in both |
| 16 | Speedometer | branch: "3.1" at `4-experiments.tex:80,107`, `1-introduction.tex:45`, caption `4-plots-original-slowdown.tex:112`, appendix; measured: 3.0. "Median 1.14x" was a distribution; PR has a Score ratio at N = 2 |
| 17 | sweep | PR 8's memcached and FFmpeg series are the verified form (8 threads 1.90 against the record cell's 1.91); marked final |
| 18 | methodology text (other branch) | "the other configurations use two builds with two runs each": Redis DE bars are 2 x 4, sweeps 2 x 1, Chromium one build and N = 3. Three branches (`app-results`, `methodology`, `fe-inl`) edit `4-experiments.tex:13-17`; none contains another |
| 19 | PR header | "State: 12 Oct", "All cells but AllOpt wait for the final record legs", "Other applications: pending": stale lines above final tables (listed in A91c, still there) |

Checked and consistent: all 36 bars of the four-panel figure, the nine MySQL write bars and the ten slowdown bars
against PR's values and rounding; AllOpt over native equals stock over native divided by the speedup in all five;
the memory reductions recomputed from the RSS triples; the site counts of the compile-time window (EXP) equal the
recount's Q_call minus the `_keep` calls in all four applications.

## 6. Not checked

- The live state of any host, leg or queue item; whether the queued items listed in the request have started.
  EXP holds no row yet for the sweeps, the Sound arms or the MySQL both-forms cell.
- Raw leg records; PR, OR and EXP were taken as given.
- Whether the Speedometer runs kept per-test scores (G8), whether the Parser and WebP stories are in either story
  sample (G1), how long one story-sample pass takes (G11), whether the third host can run the browser (G17).
- Whether the 400-unit Chromium compile-time run includes a native arm.
- The compiled PDF, page count, and how any plot looks after a swap (axis limits with negative bars or nine bars).
- Sections 2-3 of the paper beyond a grep for marked numbers (none found outside the files listed); the DE and
  bounded-shadow text (out of scope).
