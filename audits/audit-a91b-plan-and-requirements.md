# Audit A91b: camera-ready plan and requirements, completeness

Date: 14 Oct 2026 (deadline 16 Oct; team meeting 14 Oct 16:00). Read-only audit: no builds, no runs, no network.

Sources read: `plan/camera-ready-plan.md` (state 13 Oct 20:30), `atc-rebuttal.md` (rebuttal, shepherd's comment,
reviews A-E, comments @A2/@A3), `CLAUDE.md`, all live `.tex` sources at `origin/main` (78b6ceb), the diffs of
`origin/fe-inl` (0390c38), `origin/redis-results` (e5d0097), `origin/methodology` (880a7e1), `macros.tex`,
`main-atc.log` and the existing PDFs (build of 13 Oct 19:21, marks on), `plan/drafts/*`, the mapping
(`camera-ready-mapping:camera-ready-mapping.md`), `paper-results.md` (last commit 14 Oct 00:24), section 6 of
`soundness-fixes.md`.

Section numbers of the paper as built: 1 Introduction, 2 Background, 3 Sufficient instrumentation, 4 Analyses,
5 Redundancy elimination (DE), 6 Transformations, 7 Pipeline, 8 Evaluation (8.1 RQ1, 8.2 RQ2, 8.3 RQ3),
9 Related work, 10 Conclusion. "Section 8.2" of the shepherd's comment is the paragraph "Race Detection" in
`4-experiments.tex`.

## 0. Verdict in ten lines

1. Of the shepherd's six requirements, none is complete in `origin/main` or the open PRs. S6 is about half done,
   S3 has one sentence in an open PR, S1/S2/S4/S5 have no text.
2. The plan covers S1-S6 by item, but 14 concrete commitments (section 1.4) are not in it, among them: the
   rebuttal's specific preservation numbers no longer match what was measured; the "Sound" (four analyses)
   configuration promised as a separate row has no measurement outside Chromium; "at least ten runs" is not met for
   the removal form of DE (5 runs on memcached and MySQL); the remark in `3-approach.tex:102` states the opposite of
   the rebuttal's bounded-shadow argument and is outside the file the plan names for item 5.
3. Five status marks of the plan are out of date (section 2.1); the most consequential: the Redis PR carries 1.52x,
   the final figure is 1.47x.
4. The largest risk is not a number. Three sentences of the paper become false with the adopted configurations:
   "we rely only on sound static analyses" set against monitor-and-void approaches (`6-related-works.tex`, last
   paragraph, unmarked), "fully automatic" (abstract, introduction), and "avoiding LTO ... per compilation unit"
   read together with the 1-18 % compile time the committee named as a strength.
5. The wording queue is the critical path: 41 `\staleclaim` spans, each needing one approval and two reviewer-agent
   passes, in about 48 hours.
6. Page budget: the body with acknowledgments ends on page 13 and the references start there. S1, S2 and S4 add
   text; nothing is yet named to be cut.
7. Unmarked stale material exists (section 3.4): the ReX paragraph and table, the oracle-tracer table,
   "Speedometer 3.1" (the measured tree has only Speedometer 3.0), the upstreaming sentence, the memory text.
8. Front matter: no CCS concepts, no keywords, 18 of 20 floats without `\Description`, acknowledgments not in the
   class's `acks` environment, no artifact or data-availability statement of any kind.
9. Measurements still outstanding for figures the paper has today: Chromium native build (slowdown, compile time),
   DMIR for every application, FFmpeg/SQLite record cells with DE in removal form, per-site preservation rates.
10. What could not be checked is listed in section 6.

---

## 1. Commitments

State column: D = done in `origin/main`; PR = in an open PR; P = partly; N = not started.
Depends: M = measurement, W = wording approval, C = the owner of the DE and bounded-shadow text, X = mechanical.

### 1.1 The shepherd (S1-S6)

| id | commitment | where in the paper | state | depends | plan |
|---|---|---|---|---|---|
| S1 | Bounded shadow state analysis | `3-approach.tex:95-103` (remark), `3b-redundancy.tex` (new subsection or paragraph), possibly an appendix section for the adversarial programs | N. The remark still says the arguments "also apply under a weaker requirement satisfied by production tools: omitting instrumentation for accesses that cannot race must not cause the tool to miss races it otherwise reports"; the rebuttal says the opposite for TSan (victim chosen by event count modulo four) | C, W; M if the rebuttal's adversarial figures are re-taken on the camera-ready compiler | item 5 (names only `3b-redundancy.tex`) |
| S2 | Full preservation protocol in Section 8.2 | `4-experiments.tex:158-164` (two `\staleclaim` sentences today) | N. Data exist in `paper-results.md` sections 6-7, but they are not the rebuttal's data (see 1.3) | W, M (gaps in 1.3), page budget | item 7 |
| S3 | Run counts and variance throughout | methodology paragraph; every bar chart (no error bars today); every number in 8.1-8.3; the appendix figures | P. One sentence in PR #17 (builds, runs, "95 % bootstrap intervals"); no interval appears in any figure or sentence; Chromium (N = 3), the thread sweep (2 x 2), compile time (medians of 3) and preservation (10, 7 or 5 runs) are not stated | W, X (error bars in pgfplots), decision on form | items 2, 7 |
| S4 | DE usability discussion | `3b-redundancy.tex:249-258` (subsection "Trade-off Between Performance and Report Granularity") and/or 8.2 | N. The current text still offers "can be mitigated by a post-processing step" that reviewers C and E noted is not implemented | C, W; M for the relocation count (1.3, R9) | item 7 (the plan files it under the results section; the text it must change is in `3b-redundancy.tex`) |
| S5 | Sharpened positioning versus prior work | `6-related-works.tex` "Hybrid Techniques", `1-introduction.tex:59-62` | N | W; bibliography (1.3, R11) | item 9 |
| S6 | Appendix restructuring | `appendix/*`, Section 9 | P. Done: peeling moved into Section 6 (055cd7d), notation of 2.2 moved (8f6c2f3), `func_B` typo. Open: grouped citations of Section 9, appendix formatting | W (for Section 9), X | items 9, 10 |

### 1.2 Reviewers

| id | request | where | state | depends | plan |
|---|---|---|---|---|---|
| C-1 | Preservation experiment: named racy corpus, many repeated runs, detection rates with variance, sound-only separated from AllOpt | 8.2 | N (= S2). "Detection rates with variance" exist for SQLite (Wilson intervals) and Chromium; for the other applications only "k of n runs" | W, M | 7 |
| C-2 | Interaction of reduced instrumentation with bounded shadow state | Section 3 remark, Section 5 | N (= S1) | C, W | 5 |
| C-3 | Comparison with prior hybrid detectors, or explicit scoping of the contribution as engineering | Sections 1, 9, 10 | N. Nothing in the paper scopes the contribution; the rebuttal chose the "delineate differences" route | W | 9 (the "explicit scoping" alternative is not mentioned) |
| C-4 | Run counts and variance throughout | 8 | P (= S3) | W, X | 2, 7 |
| C-w2 | "DE dominates and is least novel" | 8.1, grey box at the end of 8.3 | Changes by itself: DE is no longer the strongest single configuration anywhere (1.11x at most, Redis). The text still says "DE is consistently the most powerful single optimization" three times (`4-experiments.tex:87`, `:189`, appendix `contention.tex`) | W | 2 (as red claims) |
| C-w3 | Proofs assume an idealized detector | Section 3 remark | N (= S1) | C, W | 5 |
| D-major | Main text self-contained; appendix must not bypass the page limit | Sections 2, 6, appendix | P. Notation and peeling done. Still pointing into the appendix from the main text: about ten `\cref{app:...}` (proofs, EA details, contention, oracle tracer, ReX details, the UAF premise). The contention study and the ReX numbers are evaluation content that lives only in the appendix | W | 10 |
| D-minor-1 | Grouped references in related work (up to 12) | `6-related-works.tex`: groups of 7 (predictive), 8 (sampling), 7 (static, other), 12 (static detectors) | N | W | 9 |
| D-minor-2 | Appendix formatting | appendix | N; the review does not say what. Candidates found: the DynSTC proof (`app:stc-proof`) is a subsection of "Proofs for Dominance-Based Elimination"; `lo-proof.tex` and `ea-appendix.tex` start with `\subsection` and depend on the `\section` opened in `concrete-semantics.tex`; `rex-details.tex` also holds the oracle tracer; "Details and Proofs for the Analysis" (singular) | X (structure), W (titles) | 10 (no detail) |
| D-minor-3 | `func\_B` typo | appendix C.4 | D | | struck in the rebuttal |
| E-w1 | Remaining overhead is still high (16.8x for Chromium) | 8.1 baseline, figure of slowdowns | The new figures are worse for two applications: SQLite 5.43x to 4.40x (submitted 2.4x to 1.4x), MySQL 10.03x to 9.11x. Chromium: native build pending | M (Chromium), W | 2 |
| E-w2 | DE usability trade-off not evaluated | Section 5.3 | N (= S4) | C, W | 7 |
| E-q1 | Which benchmarks, how many races, identical or equivalent | 8.2 | N (= S2) | W | 7 |
| E-q2 | Why the dominance argument holds for bounded metadata | Section 5 | N (= S1) | C, W | 5 |
| E-c | "The suggested post-processing solution is not implemented or evaluated" | `3b-redundancy.tex:257-258` | N. The sentence must either go or say that it is not implemented | C, W | not in the plan |
| B | Work is applicable to systems software, not specific to it | framing | No commitment made | | |
| A | none | | | | |

Committee strengths that the paper must not silently lose (review C strengths 1 and 3, comments @A2/@A3, the
shepherd's first sentence): deployment at Chromium and MySQL scale; no annotations; LTO avoidance; upstreaming;
"SQLite drops from 2.4x to 1.4x slowdown". Each of these is touched by the new results (section 4).

### 1.3 Promises of the rebuttal

| id | promise or figure in the rebuttal | where it must appear | state against the current data | depends | plan |
|---|---|---|---|---|---|
| R1 | "we summarize them here and will include them in the revision": the bounded-shadow arguments for all analyses | Sections 3, 5 | N | C, W | 5 |
| R2 | TSan does not evict by age; four cells; victim by event count modulo four; TSan's own intra-block elision already has the effect | Section 3 remark or Section 5 | N | C, W | 5 |
| R3 | Adversarial program for the analyses other than DE: 75 % against 74-76 %, 1000 runs per build | Section 5 or a new appendix section | N. Taken on the compiler of the rebuttal; whether it is re-taken on the camera-ready compiler is undecided | C, M? | not in the plan |
| R4 | Adversarial program for DE: 0 of 236, 236 of 236 against 165 of 236, 0.93 against 0.91 reports per run; "the same holds for post-dominance" | same | N, same question. The paper now measures DE in removal form (decided 13 Oct), so this is the form the argument is about | C, M? | not in the plan |
| R5 | Per-granule eviction counters on SQLite and memcached: same rate within 3-7 %; no racing write evicted uncovered on SQLite | same | N; not re-taken with the adopted lines (LO marks, owner marks change shadow use: LO-MARK premise M2 "a retired marker keeps three record cells instead of four") | C, M? | not in the plan |
| R6 | Configurations evaluated for preservation: Original, each of STC, SWMR, LO, EA, DE, **Sound (STC+SWMR+LO+EA)**, AllOpt (Sound+DE+Peeling) | 8.2 | P. Singles and adopted lines measured for the five applications; a combined Sound arm exists only for Chromium ("the four analyses"). AllOpt is no longer "Sound+DE+Peeling": it is a different line per application (FE-INL, inline test, run-time forms) | M (Sound arm) or W (drop the row and say why) | not in the plan |
| R7 | Criterion: identical = both stacks with file:line; equivalent (Section 5) = line of the representative access may differ within the function | 8.2 | N in the paper. The results file reports "site kept / lost / undetermined" per site and, for MySQL singles, "kept with the same lines / with other lines"; it does not report identical-versus-equivalent unions for the adopted lines | W; small M (re-reading existing reports at both levels) | 7, without this detail |
| R8 | Regression suite: 269 tests (105 race, 164 no report), all configurations identical to Original | 8.2 | Does not match. Now 391-445 tests run per root; single analyses and FE-INL clean; DE with the inline test and three adopted lines have failures (FFmpeg 12, Redis 2 in the removal form; 19-20 for SQLite and memcached in the verified form), none a missing report, most from a link defect on unoptimized units whose fix is in no measured root | W (how to state it); M if a root with the fix is run | 7, without this detail |
| R9 | Applications: union of races across **at least ten runs** per configuration; Sound identical, AllOpt equivalent for SQLite, memcached, MySQL, Chromium; Redis and FFmpeg report nothing | 8.2 | P. Ten runs hold for singles and for the verified form; the removal form, which is the form the paper now reports, has 5 runs per arm on memcached and MySQL. Redis and FFmpeg are now covered by seeded races (11 and 8-9 seeds), which is more than the rebuttal said and has to be described. Chromium is reported per race, with rates far from stock's for two of three races (crashpad 10 of 19 against 33 of 33; PartitionAlloc 2 of 19 against 11 of 21), status "current" | M (5 more runs), W | not in the plan in this detail |
| R10 | "only one report was relocated (memcached, 1 of 30 runs), and a relocated report always names the same function" | Section 5.3 (S4) | Not re-established for the camera-ready lines. MySQL singles show "kept with other lines" 1 each for SWMR, EA, DE+Peeling, and one DE pair with read and write roles exchanged | M (count from existing reports), C, W | not in the plan |
| R11 | "We will add some further discussion ... to more cleanly delineate the differences"; the SVF-based approach relies on whole-program pointer analysis that "takes hours ... does not finish within 12 hours on MySQL or Firefox" (two sources cited) | Section 9, Introduction | N. Neither source is in `references.bib` (no key matches); the related-work text has only the footnote about unavailable code | W; X (two bib entries) | 9, bibliography not mentioned |
| R12 | "our per-translation-unit analyses compile MySQL, FFmpeg and Chromium with 1-18 % compile-time overhead" | 8.2, abstract | Now MySQL +25 % over native (+16 % over stock), FFmpeg +38 % (+17 % over stock), Chromium +10 % CPU over stock (native pending). For exactly these three the statement survives in the form "over stock TSan, 10-17 %"; Redis (+189 %), memcached (+138 %) and SQLite (+69 %) do not fit, and Redis and memcached are not per-translation-unit | W (decision in the plan's list) | open decision |
| R13 | Replace the grouped citations of Section 9 with focused discussions | Section 9 | N | W | 9 |
| R14 | Fix the appendix formatting | appendix | N | X | 10 |
| R15 | "reporting run counts and variance for every experiment" | 8, appendix | P (= S3) | W, X | 2, 7 |
| R16 | Position on a user study: none; argument that iterative debugging cost is low | Section 5.3 | N | C, W | 7 |
| R17 | Peeling into Section 6; notation of 2.2 to the appendix; `func_B` | | D | | |

### 1.4 Commitments the plan file does not cover

1. **The remark in `3-approach.tex:102`** contradicts the rebuttal and must change with S1. Item 5 names only
   `3b-redundancy.tex`. Whoever owns the DE and shadow text has to own this sentence too, or it is orphaned.
2. **The rebuttal's adversarial and eviction-counter figures (R3-R5)**: reuse as measured, or re-take on the
   camera-ready compiler. Nobody is assigned; no measurement is queued in the plan.
3. **The "Sound" configuration (R6)**: promised by name, with a result ("identical union"). No arm for the five
   applications.
4. **Ten runs (R9)** for the removal form on memcached and MySQL.
5. **Identical versus equivalent (R7)** as the stated criterion: the data are per site, not per criterion.
6. **The regression-suite sentence (R8)**: the count 269 and "all identical" are both gone.
7. **The relocation count (R10)** that the usability argument rests on.
8. **The post-processing sentence** in `3b-redundancy.tex:257` (review E).
9. **Bibliography for S5 (R11)**: two entries missing.
10. **"Explicit scoping as engineering" (C-3)**: a decision whether the paper says it.
11. **Error bars or intervals in the figures (S3)**: the plan says "in the results file for every figure"; no
    item says how they reach the plots (pgfplots error bars, or intervals in a table), and this changes figure
    height and so the page budget.
12. **The workloads changed** (`paper-results.md` line 9-10: "the text must say so"): SQLite stress2 instead of
    the suite, MySQL insert/update/delete instead of select-random-points and write-only (two subfigure titles and
    labels), Redis seven data-heavy commands with 8 I/O threads, memcached pipelined 32-key gets, FFmpeg four
    transcodes, Chromium two story samples. Covered only implicitly by "all numbers".
13. **Front matter and class requirements** (section 3.6): CCS concepts, keywords, `\Description`, `acks`.
14. **Artifact / data availability**: no statement in the paper, no plan item (section 3.5).

Also absent from the plan, though they follow from its own item 2: the unmarked stale material of section 3.4 and
the new costs the paper does not report today (text size up to 2.58x on Redis, 3.0x on Chromium; Chromium peak
memory 1.54x; the V8 debug-check deaths of AllOpt, 3 of about 46 runs).

---

## 2. The plan file

### 2.1 Status marks out of date

| place | plan says | repository and results say |
|---|---|---|
| item 7 | "`methodology`, not pushed" | `origin/methodology` exists at 880a7e1 (PR #17, open) |
| item 2, Redis row | "Redis replaced (`redis-results`, pushed)"; 🟧 "AllOpt, DE, DE+Peeling provisional" | Final since the evening of 13 Oct: AllOpt 1.47x [1.45, 1.49] with DE in removal form, DE 1.11, DE+Peeling 1.08, AllOpt over native 4.22x. The PR still shows 1.52, 1.06, 1.07, 4.1 with `\prov` and a `\provnote`. The swap is a number change and needs no approval; the second Redis sentence ("reaches 1.52x only by combining ...") repeats the number |
| results table, memcached | "screening running, record cell to follow"; in the paper 🔴 | Record cell final (13-14 Oct night): 1.91x [1.90, 1.92], 4.44x over native. Only the DE and DE+Peeling columns in removal form are still queued |
| results table, MySQL | DE in removal form "tonight" | Timed 13 Oct evening: removal 0.994 of the verified form; preservation done (5 runs) |
| results table, Redis | "record cell running" | done |
| item 3 | "the run-time forms of LO and EA are not described yet" | still true; but FE-INL is now described in PR #15, and the item's "where" omits `3a-3-lo.tex` and `3a-4-ea.tex`, where the mapping puts these forms |
| progress bar | 3.5 of 10 | there are 11 items |
| S3 | "in the paper for Redis only" | The Redis PR adds no run count or interval; the only S3 text is the sentence in PR #17 |
| header | state 13 Oct 20:30 | results moved until 14 Oct 00:24 |

Not out of date but worth stating: the three PRs merge without textual conflict in any order (checked with a
three-way merge of each pair); `redis-results` changes `macros.tex` (adds `\prov`, `\provnote`, `\provfigstyle`,
`\updfigstyle`), so the marks-off check must be repeated after it lands.

Stale outside the plan: the mapping's table of adopted figures (SQLite 1.19x, FFmpeg 1.29x, Redis 1.54x, MySQL
1.13x, memcached 1.80x) against the results file (1.23x, 1.27x, 1.47x, 1.10x, 1.91x); the header of
`paper-results.md` ("State: 12 Oct"); the second compile-time table of `paper-results.md` (Redis 1.49x, FFmpeg
1.24x); `CLAUDE.md` "Known Open Issues" (the `\ignore{}` block it names no longer exists).

### 2.2 Items with no owner or no next step

No item of the plan names an owner. The fixed decisions give one for item 5 and S4; the rest is implicit.

| item | missing |
|---|---|
| 5 / S1 | Owner known; no draft, no target length, no date. The page budget (item 6) cannot be settled before its length is |
| 7 (S4 part) | Same owner as item 5 by decision, but the plan files it under the results section; make it one hand-off |
| 7 (S2 part) | No draft. Blocked by the gaps R6-R10 and by the page budget |
| 8 Conclusion | Nothing but "three numbers are red"; "without compromising soundness or completeness" (`7-conclusion.tex:6`) is not marked and depends on item 11 |
| 9 Related work | Not started, no owner, no list of which citation groups are split, no bibliography step |
| 10 Appendix | "the rest not started": no list of what "self-contained" requires (section 3.7 gives one) |
| 11 | "to be settled separately": no date; it blocks the abstract, the methodology's definition of AllOpt, Section 9's last paragraph and the conclusion |
| 6 Page limit | State only; no candidate cuts |
| Chromium | "its three figures need a new design": no owner, and the native build is pending |
| thread sweep | SQLite has no series; the figure has a SQLite panel and the text names SQLite |

### 2.3 Dependencies

- Item 11 (how the run-time forms are described) -> abstract, introduction ("fully automatic", "By default ...
  without ... runtime filtering logic"), methodology (definition of TSan+AllOpt, `4-experiments.tex:16`),
  Section 9 last paragraph, conclusion, and S2 (a voided run is an outcome the protocol must name).
- Decision on the base (upstream stock or the root's own stock) -> every speedup in item 2 (2-5 % apart;
  memcached 1.91x or 1.82x; Redis 1.47x or 1.43x; the maximum in the abstract).
- Decision on SQLite's metric -> SQLite paragraph, its bar, and whether SQLite stays in the slowdown figure in the
  present form.
- Decision on how instrumentation reduction is presented -> abstract ("up to 65 %", "23 and 58 %"), the metric
  table (SIR, DIR, DMIR), the three-panel figure, the two paragraphs under it, the memory paragraph.
- S1 and S4 text (length) + S2 text + error bars (S3) -> item 6 (cuts) -> final layout.
- Item 5's decision (DE measured in removal form) -> the preservation runs of R9 and the adversarial data of R4.
- Chromium native build -> slowdown figure, Introduction's motivating numbers (3.65, 0.19, 19.2x), compile time.
- `redis-results` merged -> marks-off build check (new macros).
- Everything -> `\finaltrue`, page count, final read.

### 2.4 Critical path to 16 Oct

1. 14 Oct 16:00: the decisions of section 5 (D1-D7). Without them roughly 30 of the 41 red claims cannot be
   drafted.
2. 14 Oct evening: S1 and S4 drafts with a length; in parallel, mechanical number swaps for Redis (final),
   memcached, MySQL, FFmpeg, SQLite and their plots; missing measurements started (ten-run preservation for the
   removal form, Chromium native build).
3. 15 Oct morning to evening: wording proposals in batches (per paragraph, not per sentence), reviewer agent,
   approval; S2 paragraph; Section 9; page cuts.
4. 15 Oct night: marks off, page count, front matter, bibliography, appendix.
5. 16 Oct: final read, upload, forms.

The binding constraint is step 3. 41 `\staleclaim` spans (abstract 4, introduction 3, evaluation 32, conclusion 1,
appendix 1) at one approval each, plus the new text for S1, S2, S4, S5 and item 11. If each approval takes ten
minutes of the approver's time the queue alone is about seven hours. A batching rule (one approval per paragraph)
should be agreed at the meeting.

---

## 3. Mechanical camera-ready requirements

### 3.1 Page limit

- `main-atc.tex` states no limit; the plan states 13 pages for the body. The venue's rule was not checked
  (no network): whether it is 13 pages excluding references, and whether acknowledgments count.
- Existing PDFs (marks on; `main-atc.pdf` and the four PR builds in `plan/drafts/`): 23 pages; the conclusion is on
  pages 12-13, acknowledgments in the right column of page 13, the references start below them on page 13, the
  appendix follows the references. So the body fills 13 pages with about a third of the last column free. The plan
  reports the same with the marks off; that build was not repeated here.
- Expected growth: S1 (a paragraph to half a column), S2 (protocol, at least a third of a column, more with a
  table), S4 (a paragraph), item 11 and FE-INL text, S5 (focused discussion is longer than grouped citations),
  error bars (taller plots), CCS concepts and keywords on page 1 (several lines). Expected shrink: none decided.
- Candidates for cuts, for the meeting, none requiring new content: the hardware-reordering paragraph and its
  figure in Section 7; the ReX paragraph (to one sentence; the data are old, see 3.4); the DMIR panel if DMIR is
  not re-measured; one of the Chromium figures (they need a redesign anyway); the oracle-tracer paragraph.

### 3.2 Marking macros

- `macros.tex`: `% \finaltrue` is commented out; it must be enabled. Marks at `origin/main`: `\stale` 58 uses,
  `\staleclaim` 41, `\upd` 3, `\stalefig`/`\stalefigstyle` 18 in six plot files and one appendix file.
- With `\finaltrue` the old numbers print as normal text. The switch hides marks; it does not prove the numbers
  were replaced. Before the final build the count of `\stale{`, `\staleclaim{`, `\stalefig`, `\prov{` and
  `\provnote{` in the live sources must be zero, and the macro definitions should then be reduced to pass-through
  or removed.
- `packages.tex:38` loads `todonotes` enabled and defines five per-person comment macros (initials in macro
  names). No live use in the camera-ready sources. Recommended: `\usepackage[disable]{todonotes}` or removal, so a
  stray `\todo` cannot print.

### 3.3 Commented-out and leftover blocks

- `4-experiments.tex:18-19`: commented post-dominance ablation with stale figures and a `\todo`. Harmless in the
  PDF; sources go to the publisher. Remove.
- `1-introduction.tex:2-7`, `:34-35` (commented `\todo`), `:110` (`% \ucomment{check.}`); `7-conclusion.tex:21`
  (commented future-work paragraph on report enrichment; it is the post-processing idea that reviews C and E asked
  about, see S4); long commented passages in `6-related-works.tex`; commented alternative titles and an anonymous
  author line in `main-atc.tex`.
- `\ignore{}`: defined in `macros.tex`, used nowhere in the live sources. The Data Availability block that
  `CLAUDE.md` names is gone.
- Live `TODO`/`FIXME`/`XXX`: none outside comments.
- Repository contents not part of the paper: `OOPSLA/`, `ECOOP/`, `ForATCSubmission/`, `extended-abstract/`,
  `upstreaming/`, `EscapeAnalysis.cpp`, `_minted-main-atc/`, build products (`main-atc.aux`, `.log`, `.pdf` are in
  the working tree). If the publisher takes a source archive, it should be built from the live file list only.

### 3.4 Stale material that carries no mark

| place | text | why stale |
|---|---|---|
| `4-experiments.tex:172-180`, `appendix/rex-details.tex` (table: 385M, 168M, 12M accesses; 99 %, 96 %, 37 %) | ReX comparison | Measured with the submitted compiler on the submitted workloads. Also: the inline test (N1) is itself a dynamic redundancy filter, and the mapping calls it the answer to this comparison; "it avoids the high per-access runtime cost of dynamic filtering, but at the expense of being more conservative" no longer describes the system |
| `4-experiments.tex:168-170`, `appendix/rex-details.tex` (oracle tracer, table of imprecision causes) | old workloads | same |
| Introduction, 8.1, caption of the slowdown figure, appendix subfigure title | "Speedometer 3.1" | The measured tree has Speedometer 3.0 only (`paper-results.md` section 1) |
| abstract last sentence, introduction, conclusion | "Our optimizations have been accepted by the TSan maintainers and are in the process of being upstreamed" | The figures now come mostly from forms that were not part of that agreement (run-time ownership and lock marks, parked threads, a modified runtime). Whether the sentence still holds as written was not checked |
| `4-experiments.tex:145-146` | "Instrumenting fewer accesses lowers memory use ..." | Memory overhead now rises for four of five applications (-0.3 to -3.2 %; memcached +10 % reduction) |
| `3bb-transformations.tex:108` | "The goal of almost-zero overhead for this dynamic filter sets us apart from ... ReX" | See first row |
| `3c-pipeline.tex:20`, `:27` | "In the final stage, the DE pass is executed"; "Soundness of the overall pipeline follows from soundness of each individual analysis" | The pipeline figure and order do not show FE-INL, the inline test or the whole-program summary step used by Redis and memcached |
| `example-prog.tex` caption | "all five optimizations" | fine as is; listed because the mapping adds forms the example does not show |
| metric table in `4-experiments.tex` (CTO relative to native; SIR/DIR/DMIR) | definitions | depend on the open decisions on compile time and on instrumentation reduction |

### 3.5 Data availability, artifact statement, badges

- The paper has no data-availability or artifact section and no badge.
- `plan/drafts/hotcrp-ae-2026-09-22.md` is a registration text asking for Available and Functional. Session notes
  record that the artifact submission was stopped on 23-24 Sep; whether an artifact was evaluated, and with what
  result (decisions were due 14 Oct), could not be established from the files read.
- Consequences to decide: (a) if no badge was awarded, nothing is mandatory, but a paper whose strengths are
  "deployment" and "upstreaming" with no pointer to code will be noticed; (b) any statement that names a
  repository must name one whose compiler produced the camera-ready figures, which the public artifact of
  September did not (five different compiler roots are in use, `paper-results.md` section 1); (c) the plan has no
  item for this; `CLAUDE.md` action 7 is still "todo".

### 3.6 Copyright, front matter, authors, acknowledgments

- Rights block: present (`\setcopyright{cc}`, `\setcctype{by}`, conference, DOI 10.1145/3828523.3857050, ISBN,
  year). The first page of the built PDF shows the CC BY 4.0 line, DOI and ISBN. Not checked against the rights
  form.
- **No `CCSXML` / `\ccsdesc` and no `\keywords`** anywhere in the live sources. The class normally expects both in
  a final version; whether this venue requires them was not checked.
- **`\Description`**: 2 of 20 floats have one (one in `4-plots-speedup.tex`, one in `appendix/chromium-selected.tex`). Missing: `example-prog.tex` (1), `3a-4-ea.tex`, `3b-redundancy.tex`, `3c-pipeline.tex`, `4-experiments.tex` (metric
  table figure), `4-plots-contention.tex`, `4-plots-instr-reduction.tex`, `4-plots-original-slowdown.tex`,
  `4-plots-overheads.tex`, two of three in `4-plots-speedup.tex`, one of two in `appendix/chromium-selected.tex`,
  the appendix figures and tables. The one present in `4-plots-speedup.tex:222` describes the old MySQL panels
  ("Select and Write-only") and old values.
- `\pagestyle{plain}` and `printfolios=true` are submission settings; whether the final version keeps page numbers
  was not checked.
- Author block: five authors with affiliations and e-mail addresses; `\shortauthors` set. To confirm by the
  authors: list, order, affiliations, and that every address is the one to publish.
- Acknowledgments: written as `\paragraph{Acknowledgments}`, not the class's `acks` environment; the grant number
  is typeset in angle brackets (`<NRF-NCR25-Fuzz-0001>`), which looks like an unfilled template; no thanks to the
  reviewers or the shepherd (optional, customary).
- `minted` is used for listings (needs shell escape and Pygments); relevant only if the publisher compiles the
  sources.

### 3.7 References, labels, bibliography, appendix

- Labels: every `\cref`/`\Cref` target in the live sources at `origin/main` has a label (checked by extraction
  with comments stripped); the build log of 13 Oct has no undefined-reference warning. Not re-checked on the PR
  branches beyond their diffs, which add no reference.
- Labels that new text will need and that do not exist: a label for the bounded-shadow discussion (S1), for the
  preservation protocol (S2, so that Section 5.3 and the abstract's claim can point to it), for the FE-INL and
  inline-test paragraphs (unnumbered `\paragraph`s today), for a statement of premises if one is added.
- Citations: 64 keys cited, all present in `references.bib` (86 entries; 22 unused, harmless).
- Bibliography that a sharpened Section 9 needs: the two SVF scalability sources quoted in the rebuttal (R11) are
  missing. Present already: the Java hybrid detectors, RedCard, BigFoot, the SVF-based TSan work, the
  section-based analysis, ReX, the Go-at-scale study, the driver-locking hybrid work. If the run-time forms are
  described as ownership transfer or run-time checked lock ownership, the paper has no citation for either idea
  (the only neighbour cited is the optimistic hybrid analysis, in a sentence that distances the paper from it).
- Appendix self-containment (item 10):
  - The appendix recalls main-text statements by reference (`prop:stc-conc`, `th:stc-dyn`, rules (1)-(4) of
    `ss:ea`, `def:post-domination` "as explained in the main text", `sec:sufficient`). All targets exist; the
    notation figures now live in the appendix and Section 3 points to them (`3-approach.tex:25`), which is the
    direction review D objected to, reduced to one sentence.
  - `th:stc-dyn` and its proof cover the thread counter. The merged sentence on parked threads
    (`3bb-transformations.tex:109`) has no argument anywhere; the PR text that introduced it says so ("Left
    open"). A reader of the appendix finds a theorem that does not cover the configuration that carries Redis.
  - Nothing in the appendix covers: the run-time forms of LO and EA (premises in `soundness-fixes.md` 6a and 6f),
    the inline test, FE-INL, post-dominance premises (the plan's open decision on forward progress and A9), the
    bounded-shadow analysis, the preservation tables.
  - `appendix/contention.tex` and `appendix/chromium-selected.tex` hold only stale data and stale claims; the new
    sweep has no SQLite series, and for memcached the gain falls as threads are added.
  - Structure: see D-minor-2 in section 1.2.

### 3.8 Terminology after the PRs

| term | where it diverges |
|---|---|
| number of optimizations | Abstract and introduction: four analyses + DE + one dynamic technique; conclusion: "five complementary static analyses ... and a transformation". Section 6 after the PRs has four things (peeling, DynSTC, inline test, FE-INL); the evaluation says "three variants from Section 6" |
| AllOpt | Defined as EA, LO, STC, SWMR and DE (`4-experiments.tex:16`, red). It is now a different line per application; FE-INL is in it for Redis, MySQL and Chromium only; peeling for SQLite |
| "four configurations" | The sentence counts Native, Original, TSan+<Opt>, AllOpt, then adds three |
| DynSTC | Section 6 defines it as a branch on a thread counter; the Redis column is parked threads with a phase guard; FFmpeg's is a run-time option plus compiled guards |
| DE | Section 5 describes removal; decided 13 Oct that it is measured so. Consistent, provided no figure taken with the verified form remains (FFmpeg's record cell is "DE in its verified form"; SQLite's record leg likewise) |
| "inline test" | Introduced in Section 6 as a paragraph without a configuration name; the Redis PR sentence uses "the inline test" as a component of AllOpt. No bar for it in any figure |
| speedup base | "Original TSan" in every caption; the figures are over the compiler root's own stock build unless decided otherwise |
| "Memcached" / "memcached", "x" / "×" | mixed (`1.07x` in 8.1, `×` in the abstract and conclusion) |

---

## 4. Consistency risks created by the new results

Severity: how a reviewer who reads the camera-ready against the submitted version and the rebuttal would take it.

| # | claim in the paper | now | kind | severity |
|---|---|---|---|---|
| 1 | Abstract: "our optimizations never cause TSan to miss races it would otherwise detect"; introduction: "with a guarantee that no races are missed", "does not affect the detector's output beyond reducing its overhead"; conclusion: "without compromising soundness or completeness" | (a) The rebuttal itself concedes that any change of instrumentation changes evictions; premises P-EV, P-RESET, P-REPORT for removal-form DE. (b) In the adopted SQLite and memcached lines a violated ownership assumption ends the run with a void verdict and no race report for that access. (c) Adopted premises beyond the paper's one (no use-after-free): signals, typed indirect calls, closed world, in-bounds, strict aliasing, application-specific ones | wording decision (item 11, S1) | **high**: it is the central correctness claim, and reviews C and E attacked exactly it. The abstract sentence is not marked red |
| 2 | Section 9, last paragraph: "Unlike approaches that use optimistic ... invariants and recover soundness by monitoring violations and re-executing ..., we rely only on sound static analyses" | The memcached and SQLite lines assume an ownership invariant, monitor it at run time, and void the run on a violation | wording decision | **high**, and unmarked: the paper would criticise prior work for the design its best figure now uses |
| 3 | "fully automatic" (abstract, introduction), "places no additional burden on developers"; committee: "no annotations" | SQLite: a generated spec (mutex API and object type names; 98.7 s once per source version; a build without debug info is refused). memcached: a candidate list from a separate tool (no hand annotations: 1.80x; seven annotated fields: 1.81x). Redis: whole-program summary, external-symbol list for `deps/`. Both run-time forms were built for and measured on one application each | wording decision | **high**: named strength of the acceptance. Defensible if the text says what input each needs and that it is derived, not written by hand; not defensible as "fully automatic" unqualified |
| 4 | "avoiding LTO", "per compilation unit" (Section 7), and the rebuttal's contrast with whole-program analysis | Redis and memcached use the whole-program summary step (memcached: 6.53 s of a 16.32 s build). Section 7 already describes summary files as optional, so the mechanism is in the paper; what is new is that two of the six headline figures need it and that STC, SWMR and LO remove nothing without it on SQLite, FFmpeg and MySQL | wording decision | medium-high: still no LTO in the strict sense, but "per translation unit" as the selling point weakens |
| 5 | Compile time "1-18 %", "modest", "do not exceed 18 %", "only 2.5 %" for FFmpeg | Over native: +25 % (MySQL) to +189 % (Redis); over stock TSan: +16 % to +168 %; Chromium +10 % CPU over stock. Stock TSan alone is +7 to +36 % over native on the new host, so even the baseline exceeds the old range | number swap plus wording ("modest" cannot stay); decision on the reference (native or stock) and on reporting absolute seconds (the worst ratios are builds of 5-16 s) | **high**: committee strength, quoted in the rebuttal with numbers |
| 6 | "up to 65 % of instrumentation calls being pruned"; 64.9 % and 60.8 % static | The analyses remove 0.6-12.4 % of checks; the count of checks in the binary is above stock (+5 to +27 %) because of peeling, function copies and inline forms | wording decision and a new figure; not a swap | **high**: an abstract number falls by a factor of five to fifty, and the metric's meaning changes |
| 7 | "reduces the number of instrumentation calls at runtime by between 23 and 58 %" | Runtime entries drop 12 % (SQLite) to 94.5 % (Redis); a different quantity (calls into the runtime, not executed checks), "current" status, MySQL about 1-17 % | redefinition (DIR) + swap | medium |
| 8 | DMIR (unique locations): 43.9 %, 41.3 % | No new measurement in the results file | needs-measurement or drop the panel | medium (silent removal of a metric is visible in the metric table) |
| 9 | Maximum speedup "up to 1.71x" (SQLite) | 1.91x memcached (1.82x over upstream stock); SQLite 1.23x; FFmpeg 1.57 -> 1.27; MySQL 1.16/1.11 -> 1.10; Redis 1.45 -> 1.47 | number swap; the base decision moves the maximum | low for the maximum; medium for the pattern: three of five applications fall, and the text must say that workloads and compiler changed |
| 10 | "peak speedups reaching 2.54x under high thread contention" | memcached sweep: 2.11x (clean) or 2.23x (flagged) at the **lowest** thread counts; the gain falls as threads are added. FFmpeg flat. SQLite: no series | wording decision; "under high thread contention" is now wrong in direction | medium-high |
| 11 | Chromium geometric mean 1.39x; medians 1.14x-2.16x per suite; CDF over all tests; "Speedometer 3.1" | 1.25x (hold-out sample of 48 stories; 1.29x on the first sample), N = 3, Intel; on the second host 1.02-1.08x. Speedometer 3.0: 1.225 (N = 2). AllOpt died in 3 of about 46 Speedometer runs on a V8 debug check; peak memory 1.54x; text 3.0x | swap + new figure design + wording (two hosts disagree; which one the paper reports is decided, but stating only the favourable host is a risk) | medium-high |
| 12 | "SQLite shows the highest application-level speedup, reaching 1.71x ... clear synergistic effect"; review C strength 3 (2.4x -> 1.4x) | 1.23x on one stress test; slowdown 5.43x -> 4.40x; the gain disappears under a counting profiler and at 2 CPUs; successful data-modifying statements fall (0.75-0.88x) while trivial ones rise | wording decision (metric to be named; the results file says the paper must state that successful data-modifying statements fall) | **high** if stated as a plain speedup; low if the metric and its limits are named |
| 13 | "DE is consistently the most powerful single optimization" (three places), grey box | DE alone: 0.98-1.11x; the gains are carried by run-time forms and FE-INL | wording | medium; it also removes review C's "DE dominates" objection, which Section 9 can use |
| 14 | "reduction in memory overhead for almost all programs" | an increase for SQLite, Redis, MySQL, FFmpeg; 10 % reduction for memcached | wording | low-medium |
| 15 | Baseline slowdowns 2.4x-19x; I/O-bound versus in-memory classification of applications | 2.76x (FFmpeg) to 10.03x (MySQL); SQLite 5.43x and memcached 8.48x are no longer "moderate ... as I/O latency masks" | swap + wording | medium |
| 16 | "By default, we do this without introducing any additional runtime filtering logic" (introduction, red) | Most adopted gains are run-time tests | wording | medium (already marked) |
| 17 | "Each configuration was run multiple times"; new sentence of PR #17 | PR text matches the record legs (4 builds x 4 runs; singles 2 x 2). Not covered by it: Chromium N = 3, Redis DE cells 2 x 4, MySQL built `-O2 -g`, Chromium component build with DCHECKs, five different compiler roots across applications, legs run with the other half of the host busy | wording | low-medium; the compiler-root point is the one a careful reader would ask about |
| 18 | "accepted by the TSan maintainers ... being upstreamed" | see 3.4 | wording | medium, unverified |
| 19 | Regression suite and preservation sentences of the rebuttal | see R6-R10 | wording + small measurements | **high** relative to the shepherd: S2 asks for "the full preservation protocol" as promised, and the promised figures are not the ones on hand |
| 20 | New costs not in the paper | text size 1.06x-2.58x (Chromium 3.0x); SQLite +51 % build | new sentence | low, but omission would be noticed with a 2.58x binary |

---

## 5. Actions for the next 48 hours

Tags: needs-measurement, needs-wording-approval, mechanical, colleague (owner of the DE and bounded-shadow text).
"16:00" marks what should be decided at the meeting.

| # | action | tag | size | 16:00 |
|---|---|---|---|---|
| 1 | Decide how the paper states its guarantee and its automation given the run-time forms (item 11): the abstract sentence "never cause ... miss", "fully automatic", the definition of AllOpt, Section 9's last paragraph, the conclusion. One decision, five places. Without it the abstract cannot be drafted | needs-wording-approval | 30 min to decide, 2 h to draft and review | **yes (D1)** |
| 2 | Hand-off of S1 + S4 + the remark in `3-approach.tex:102` + the post-processing sentence as one package to its owner, with a length cap and a time (first draft by the evening of 14 Oct); decide whether the rebuttal's adversarial and eviction figures are reused as measured or re-taken | colleague (+ needs-measurement if re-taken) | 3-4 h of writing | **yes (D2)** |
| 3 | Decide the speedup base (upstream stock or the root's stock) and state it once in the methodology | needs-wording-approval | 10 min | **yes (D3)** |
| 4 | Decide the presentation of instrumentation reduction (abstract's 65 % and 23-58 %; SIR/DIR/DMIR table and figure; DMIR re-measured or dropped) | needs-wording-approval (+ needs-measurement for DMIR) | 20 min to decide; half a day if a new figure | **yes (D4)** |
| 5 | Decide compile time and the LTO/summary wording: reference (native or stock), absolute seconds, and how Redis's and memcached's whole-program step is named | needs-wording-approval | 15 min | **yes (D5)** |
| 6 | Decide what is cut for the page limit, and the batching rule for approvals (per paragraph, not per sentence) | needs-wording-approval | 20 min | **yes (D6)** |
| 7 | Decide the artifact or availability statement, and confirm the artifact-evaluation outcome | needs-wording-approval | 10 min | **yes (D7)** |
| 8 | Update `redis-results` to the final figures (1.47x; DE 1.11; DE+Peeling 1.08; 6.21x and 4.22x over native), drop `\prov`/`\provnote`; then merge the three PRs (no conflicts found) | mechanical (the second Redis sentence repeats the number: number-only change) | 30 min | no |
| 9 | Swap the numbers and bars for memcached, MySQL, FFmpeg, SQLite in the text, the speedup and slowdown figures, the thread-sweep figure (memcached, FFmpeg), with intervals (error bars or bracketed intervals per the S3 decision) | mechanical | 3-4 h | no; needs D3 |
| 10 | Close the preservation gaps: 5 more runs of the removal form on memcached and MySQL (ten in all); a count of relocated reports and of identical-versus-equivalent unions from the reports already on disk; decide on the Sound arm (measure, or drop the row with a sentence) | needs-measurement | 2-4 h machine time on the non-timed host; 1 h reading | Sound arm: yes |
| 11 | Draft the S2 paragraph for 8.2 (corpus, configurations, runs, criterion, seeded races for Redis and FFmpeg, void runs, suite result with the link defect stated) and the S3 statements for Chromium, sweep, compile time | needs-wording-approval | 2 h | no |
| 12 | Chromium: native build for the slowdown and compile time, and the new design of its three figures; SQLite's metric named in the text | needs-measurement + needs-wording-approval | native build: hours of machine time; figures 2 h | SQLite metric: **yes** (in the plan's list) |
| 13 | Section 9 (S5, review D): split the four large citation groups into focused sentences, add the two missing bibliography entries, add the scoping sentence if decided, use the fact that DE no longer dominates | needs-wording-approval | 2-3 h | scoping: yes, briefly |
| 14 | Wording of the remaining red claims in 8.1-8.3, the introduction and the conclusion, including the unmarked stale items of section 3.4 (ReX, oracle tracer, Speedometer version, memory, upstreaming sentence) | needs-wording-approval | 4-6 h including reviews | no |
| 15 | Final mechanics: `\finaltrue` and zero remaining marks; `todonotes` disabled; commented stale blocks removed; CCS concepts and keywords; `\Description` for 18 floats and the outdated one; `acks` and the grant number; appendix structure (D-minor-2), stale appendix figures; page count; a last label and citation check | mechanical | 2-3 h | no |

Measurements that should start today regardless of the meeting, because nothing decided there removes the need:
item 10's extra runs and the Chromium native build. Both are non-timed or on the other host and do not disturb
timed legs if scheduled outside them.

---

## 6. What was not checked

- The venue's camera-ready rules: page limit and what counts toward it, whether CCS concepts, keywords, page
  numbers, `\Description`, an artifact appendix or a badge are required; the submission system and forms. No
  network access was used.
- The page count with `\finaltrue`. No build was run; the statement about page 13 comes from the PDFs built on
  13 Oct with the marks on, and from the plan for the marks-off case.
- The state of the pull requests on the hosting service (open, reviewed, approved). Only the branches were read.
- The outcome of artifact evaluation, and whether an artifact was submitted at all after 23 Sep.
- Whether the upstreaming sentence still holds.
- Any figure in `paper-results.md`, `soundness-fixes.md` or the mapping against raw data. They were read as given.
  `optimization-results.md` and `experiments.md` were only searched, not read.
- Bar-by-bar values in the plot files against the results file (only the Redis PR diff was read).
- The rebuttal's adversarial-program and eviction-counter data: where they are stored and on which compiler.
- `3a-*.tex`, `3b-redundancy.tex` lines 1-82 and 104-246, and the appendix proofs were searched for claim-bearing
  phrases and structure, not read line by line.
- The three-way merge check of the PRs covers textual conflicts only, not the combined build.
- Rights form, author metadata and funding text against any external record.
