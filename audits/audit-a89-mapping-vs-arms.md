# Audit A89: the mapping against the arms as built

13 Oct 2026. Read-only: files were read, nothing was built or run, no host was contacted.

Sources read: `camera-ready-mapping.md` (whole), `optimization-results.md` (whole), `paper-results.md` (whole),
`measurement-protocol.md` (whole), `experiments.md` (lines 1119-1137, 2514, 2540-2606, 3880-4117 and greps),
`soundness-fixes.md` 496-530, `notes/ownership/r3-phase-skip-entries.md` 1-60, `notes/ownership/redis-phase-guard.md`
1-40, `notes/ownership/phase-quiet-rt.md` 740-756, `notes/ownership/evconf-checked.md` 1-40, the `*_READY` lists in
`/extra/alexey/cov1-2026-09-27/`, and the `build_info.txt` of every core and single-configuration tree
(`trees/<code>__/tsan-experiments/...`, 51 files), the two spec files named in the adopted flag lines, and the paper
lines the mapping cites (`3a-*.tex`, `3b-redundancy.tex`, `3bb-transformations.tex`, `3c-pipeline.tex`).

Short names used below: MAP = `camera-ready-mapping.md`, OR = `optimization-results.md`, PR = `paper-results.md`,
EXP = `experiments.md`, PROTO = `measurement-protocol.md`. "Rule" = a single configuration is the paper's analysis
plus every new optimization mapped to it that the application's adopted AllOpt line uses.

## 0. Verdict

The three known holes (Redis DynSTC, memcached EA, FFmpeg DynSTC) are not the only ones. Beyond them:

- one paper-table row (FFmpeg) has two cells measured on a different configuration than its AllOpt cell;
- the single arms were built from one template, not from each application's AllOpt line, so 14 arms carry an
  optimization their AllOpt line does not use, and the paper's own DynSTC is in no arm of any application;
- the mapping can stand, but the Rule does not say what an arm contains when its optimization needs one mapped
  elsewhere: N1-ST and quiet threads (DynSTC) work through N1's inline code (DE). Without an answer the DynSTC
  column cannot show the recorded gains. Section 4 puts this as three questions; no mapping row is changed;
- MAP is stale in every adopted figure but Chromium's and still calls the adopted SQLite optimization "not adopted".

## 1. Findings, ranked

Severity: A = a wrong or mislabelled figure in the paper table or its text; B = an optimization missing from, or
misplaced in, an arm; C = a defect of the mapping; D = stale text.

| # | sev | finding | evidence | fix |
|---|---|---|---|---|
| 1 | A | **FFmpeg row mixes two configurations.** AllOpt is `fk6` since 13 Oct (N1 placed statically: `-tsan-inline-hit-test-static-hot=16 -static-min-freq=4 -static-max-insts=400`). The DE and DE+Peeling cells (1.02, 1.03) come from `f2d`, `f2p`, which carry `-tsan-inline-hit-test` at every check, i.e. the placement of `fcn`, the line that is no longer AllOpt | PR:31; `trees/f2d__/.../ffmpeg-tsan-dom/build_info.txt` flags line; `trees/fk6__/.../build_info.txt`; EXP:3933 | Rebuild DE and DE+Peeling with fk6's three static flags added (2 arms x base, 00, 16) and re-time; or keep the cells and label them "N1 at every check (the 1.29x variant)". The first is what the Rule requires |
| 2 | A | **PR text quotes fcn's figure as AllOpt's.** "AllOpt's compiled code adds 1.13 on top of the option" is 1.285 / 1.138 = fcn. For the adopted fk6 it is 1.238 / 1.138 = 1.09 | PR:50; EXP:1127, 1130 | Replace 1.13 by 1.09 for AllOpt, give 1.13 for the every-check variant |
| 3 | A | **Adopted figures that fell, and mixed bases.** MySQL 1.13 -> 1.10: the 1.13 was over pristine upstream stock (EXP:1046 "over pristine 1.134"), the final 1.101 is over the root's own stock (`zys`); OR:95 still says "MySQL's 1.13x [is] over upstream". memcached: OR section 1 adopts 1.80 (over upstream stock), PR's table cell is 1.89 (over the root's stock). So OR section 1's adopted column holds one figure over upstream stock (memcached) and four over the root's stock, and the two files disagree on memcached (1.80 vs 1.89). Redis 1.54 -> 1.52 (MAP:40, OR:283 still 1.54). SQLite 1.28 (other half idle) -> 1.23 (busy). FFmpeg 1.29 -> 1.24 by decision | OR:49, 50, 74, 76, 95; PR:33-34, 36-43; EXP:1046, 1133 | Decide the base once (PR:41-43 leaves it open) and quote every application over it. Until the MySQL base check runs, mark MySQL's fall as "base changed, not re-measured over upstream". Fix OR:95 |
| 4 | A | **Single cells that contradict OR section 3.** (a) memcached EA 1.003 [1.001, 1.005] although EA-CONTENTS (on by default, so inside `m4E`) is recorded as **+1.4**; (b) memcached DE 0.977 [0.959, 0.996]: a column below 1 with an interval that excludes 1; (c) FFmpeg DE 1.024 with N1 inside, against N1 **+5.0**; (d) SQLite EA 0.968. Root cause: section 3 is not comparable with a single arm: its rows are increments over all the paper's analyses together, the Redis and FFmpeg entries are on the Intel host (mark `f`), and several predate the current roots and workloads | PR:30-34; OR:182-207 (header and marks), OR:75; EXP:1131 | The rule of 13 Oct ("single columns show the gains of section 3") cannot be checked against section 3 as it is. Add to section 3 one line per application from the final singles legs (AMD, current root, over stock), and state that the older rows are increments on another base. Re-time memcached DE at 4 offsets before printing 0.98 |
| 5 | A | **Name collision "AllOpt" on Chromium.** OR section 2 and section 6 define AllOpt = Sound + DE + peeling (0.993). PR:69 and OR section 1 use AllOpt for CR1+N1 (1.286 / 1.248). PROTO:88 uses the second meaning | OR:113, 679; PR:67, 69; PROTO:88 | Rename OR's row to "Sound + DE + peeling" and drop "AllOpt" from OR:679 |
| 6 | B | **The DynSTC column is DynSTC-RT in an application whose AllOpt does not use it.** Arm = stock binary + `dynstc_rt=1` for SQLite, Redis, memcached (and MySQL by the template). None of these four AllOpt lines sets `dynstc_rt` (SQLite, MySQL, memcached: no run-time options; Redis: `phase_guard=1,phase_guard_ranges=1,phase_guard_stats=1` only). The paper's own DynSTC (compiled guard, config `tsan-stmt`) is in no arm and in no AllOpt line of any application | EXP:3899, 3904-3905, 3916, 3942, 3955, 3965-3966; EXP:1126, 1128, 1131 | By the Rule the cell is "the paper's DynSTC + nothing" for SQLite, memcached, MySQL: either time `tsan-stmt`, or print the DynSTC-RT figure with a footnote "not part of AllOpt here". State in the text that DynSTC-RT replaces the compiled guard everywhere |
| 7 | B | **Redis DynSTC (the requested correction) has an unresolved dependency.** Quiet threads sets the single bit of the thread's fast state (`fast_state.single = dynstc_single || phase_skip`); compiled code obeys it through N1-ST's `=fs` front test at inlined sites and through `__tsan_phase_read/write` entries at the others. `rdy` therefore carries `-tsan-inline-hit-test-skip-single=fs`, which neither MAP:40 nor OR:48 lists. Measured: the record line without the hit test (`rdn`) reads 1.283, with it 1.492; "FE-INL + N1" without quiet threads reads 1.10. So N1 is worth about +16 % under quiet threads and +2.5 to +6 % without: the interaction belongs to neither column | `notes/ownership/r3-phase-skip-entries.md`:23, 49-51; `notes/ownership/redis-phase-guard.md`:18; `trees/rdy__/.../build_info.txt`; EXP:1119; OR:80, 298 | See section 4, Q1. Two honest arms exist: (i) without N1: `-wp` config, the six `-tsan-phase-*` flags, `=fs`, the three options (expected about 1.2 without FE-INL); (ii) with N1 added, N1 named as a prerequisite. Section 3's +31.9 % was measured with N1 present, so only (ii) can show it |
| 8 | B | **FFmpeg DynSTC (the requested correction): N1-ST cannot exist without N1 and without `dynstc_rt=1`.** The corrected arm is `-tsan-inline-hit-test -tsan-inline-hit-test-skip-single=front` (+ fk6's three static flags for the AllOpt variant) with `dynstc_rt=1`. It contains N1, mapped to DE. Since the four analyses read 1.00 on FFmpeg, this arm will read about AllOpt itself | EXP:3906-3907; OR:499-507; EXP:1130 | Build it in both placements (fk6's and every-check) or only fk6's; name N1 as prerequisite (section 4) |
| 9 | B | **FFmpeg AllOpt and its DynSTC arm carry `phase_guard=1` with no phase code compiled in.** The option is the run-time half of quiet threads. On FFmpeg nothing obeys the bit (`=front`, no `tsan.phase_ctor`), and the hooks run until the self-off. The template's DynSTC arm is `dynstc_rt=1` alone; FFmpeg's was run with `dynstc_rt=1, phase_guard=1` + stats. Which of the two options costs h264 its 7 % was never separated | EXP:3926-3927, 1130 ("the DynSTC column = ... dynstc_rt=1, phase_guard=1 + stats"); `notes/ownership/phase-quiet-rt.md`:749-754; OR:72 | Either drop `phase_guard=1` from FFmpeg's adopted options (one untimed check that the binary behaves the same, then a short leg), or add an options-only arm with `dynstc_rt=1` alone to the next FFmpeg leg and name the option in OR section 1 |
| 10 | B | **EA arms carry MEMINTR that no application's AllOpt uses.** `s9e` (SQLite), `f2e` (FFmpeg), `y8e` (MySQL) have `-tsan-memintrin-one-side`; `r5E`, `m4E` do not. No application AllOpt line has the flag (only Chromium's CR1 has) | build_info of `s9e__`, `f2e__`, `y8e__`, `r5E__`, `m4E__`; EXP:3896, 4061, 4065 | Rebuild the three EA arms without the flag, or keep them and footnote. SQLite's EA 0.968 is the cell most likely to move (section 3 gives MEMINTR +2.5 there) |
| 11 | B | **SWMR arms carry SWMR-ROOTS that the AllOpt line does not.** `s9w`, `f2w`, `y8w` have `-tsan-swmr-thread-roots`; the AllOpt lines of SQLite, FFmpeg, MySQL do not. Expected inert in a unit-by-unit build; shown only for SQLite (`s9w` is byte-identical to stock) | build_info of `s9w__`, `f2w__`, `y8w__`, `z9L__`, `fk6__`, `zyb__`; `SQFIN_TREES_READY` | Compare `f2w`, `y8w` section by section with a build without the flag (untimed), or rebuild. Low risk |
| 12 | B | **FE-INL column for applications whose AllOpt has no FE-INL**: SQLite (1.02), FFmpeg (1.00), memcached (1.00). Only Redis, MySQL and Chromium use FE-INL | build_info of `z9L__`, `fk6__`, `fcn__`, `kca__` | Keep the column (it is defined as standalone) but footnote the three cells "not part of AllOpt" |
| 13 | B | **DE arms of Redis and memcached are unit-by-unit; their AllOpt lines are whole-program.** The ruling of 12 Oct covers STC, SWMR, LO, EA only. `r5d`, `r5p`, `m4d`, `m4p`: "summaries: none" | EXP:4051-4053; build_info of `r5d__`, `r5p__`, `m4d__`, `m4p__` | State whether dominance elimination reads summaries. If it does, rebuild the four arms with `-wp`; if not, record that in EXP |
| 14 | B | **On Redis and FFmpeg the DE column measures N1, not the paper's DE.** With the hit test at every check the verified form adds nothing (MAP:187-188). No arm has verified DE without N1 there, so the table cannot say what the paper's DE gives on these two. On SQLite, memcached, MySQL the DE column is verified DE without N1 (consistent with the Rule: their AllOpt has no N1) | build_info of `r5d__`, `f2d__`, `s9d__`, `m4d__`, `y8d__` | Footnote the two rows; no rebuild needed |
| 15 | B | **SQLite's AllOpt line still has DE and peeling**, against the decision of 12 Oct that the SQLite row moves to the variant without them. The adopted compile time (+51 %) is with both | `trees/z9L__/.../build_info.txt` (config `tsan-dom_peeling-ea-lo-st-swmr`, `-tsan-de-verified`); OR:90-91; PR:117 | Decide: either the decision is superseded by the adoption of LO-MARK v1 on the existing tree (then say so), or AllOpt is rebuilt without DE and peeling, and the DE, DE+Peeling cells become "not part of AllOpt" |
| 16 | B | **Arms byte-identical to stock**: SQLite STC and SWMR, Redis LO. Printed "= stock", not timed. The superseded unit-by-unit arms (three of Redis's, memcached's SWMR) were also stock | `SQFIN_TREES_READY`, `FINARMS_redis-wp_READY`; EXP:1126, 1128, 4053-4054 | None; keep the "= stock" mark and its sentence (PR:52-53) |
| 17 | B | **MySQL has no single-configuration figures on record.** The arms exist (`y8t y8w y8l y8e y8d y8p y8f`), a single-arm leg is mentioned as running on 13 Oct, no row in EXP; PR:33 has empty cells | `FINARMS_mysql_READY`; EXP:1134 ("the MySQL single-arm leg") | Add the row when read. Expect FE-INL near AllOpt: an earlier leg read the analyses alone at 0.99 and with FE-INL at 1.10 (EXP:1188), while OR section 3 records FE-INL as +3.6, another stale row |
| 18 | B | **Preservation is missing for the DynSTC column in every application** and for MySQL's FE-INL arm (the arm that carries its figure). PROTO M8 asks for every configuration shown | PR:159-179 (arm lists) | Add the options-only arm to the seeded and site runs; add `y8f` |
| 19 | C | **N1 mapped to DE is the row a reviewer will reject first**, and Chromium shows why: CR1+N1 has no DE and no peeling at all (CR1 = four analyses + SWMR-ROOTS + FE-INL + MEMINTR), yet MAP:43 reports its gain "under DE". The hit test fires on any matching record, needs neither dominance nor must-alias, and removes no check (MAP:223-224 concedes this) | EXP:2593-2595, 2547; OR:680-681; MAP:31, 43, 262 | The mapping stands (N1 under DE). Wording only: the Chromium row says "N1, the run-time form of DE; the static DE and peeling are not part of this configuration", and the DE text explains why the static rule is left out there (peeling costs 1.9 %, one V8 unit does not finish compiling under DE) |
| 20 | C | **STC or DynSTC: the mapping says both.** MAP:14-15 "(STC)", MAP:27 row "STC, DynSTC", MAP:65 heading "STC / DynSTC", MAP:39-40 "reported under DynSTC". The arms treat STC as static only | MAP:14-15, 27, 39-40, 65 | One name: DynSTC. Change MAP:15 and split row MAP:27 into STC (nothing merged) and DynSTC |
| 21 | C | **Optimizations in adopted lines with no mapping row.** (a) the whole-program summary step (`-tsan-use-analysis-summaries`, `-tsan-summary-local-globals`): Redis, memcached; it changes STC, SWMR, LO, EA results and has only a "Pipeline" line in MAP:255; (b) EVCONF-RANGES, -ARGS, -INTERCEPT, `-tsan-evconf-ranges-in-bounds`: all four flags are in `kca`, MAP:243 calls the EVCONF variants "not for the paper"; (c) the library-facts table: MAP:243-244 says "in no adopted figure", but it is on by default (the table-off build needs `-tsan-lib-facts=false`) and no adopted line passes that flag, Chromium's included; (d) the static placement rule of `fk6` (hot 16, frequency 4, at most 400 instructions) and of `rk4`: OR section 6's placement table has N1-L, K4, s1, s4, k2, S56 and none of them is fk6's rule; (e) "phase guard": the name used for the flags and options appears nowhere in MAP or OR section 6, which say "quiet threads" | build_info of `kca__`, `rdy__`, `fk6__`; EXP:2514 (cr1nolf), `soundness-fixes.md`:505-521; OR:477-487; MAP:243-244, 255 | Each goes under an old name, none becomes a column: summaries -> the analysis that reads them (STC, SWMR, LO, EA: "whole-program form"); the EVCONF sub-forms -> EA, as parts of EVCONF-CHECKED; fk6's and rk4's placement rules -> DE, with N1 (name the rule in OR section 6); phase guard -> DynSTC, as the mechanism of quiet threads (add the synonym to the terms). Library-facts table: question Q3 in section 4 |
| 22 | C | **VWIDE-loops is listed as widening a static rule and as merged into DE, and is in no adopted line.** OR marks it R (run-time test) | MAP:16, 31, 191-193; OR:195, 512-519 | It stays under DE; correct MAP:16 and the table cell to the run-time side (it is the verified form's test), and say in its entry that no adopted line uses it, so no DE arm contains it |
| 23 | C | **MEMINTR -> EA: half of the rule is not EA's.** "Source is a constant" uses no escape fact; OR section 6 files MEMINTR under the form of the instrumentation. It is in no application AllOpt, only in Chromium's, where its share was never separated from FE-INL's | MAP:30, 146-147; OR:219, 532-535 | Say "EA's fact for a private source; trivial for a constant". On Chromium print the step as "FE-INL + MEMINTR" (as PR:68 does) and do not credit EA with a number |
| 24 | C | **MySQL "carried by FE-INL, +3.6 % (its whole 1.13x configuration)"** is self-contradictory: +3.6 is not 1.13 | MAP:41, 234; OR:194 | Replace by the single-arm figure when read (finding 17) |
| 25 | C | **Quiet threads on Redis sits under a name of which nothing is present.** Redis's AllOpt has neither the compiled DynSTC guard nor `dynstc_rt`; the corrected DynSTC arm will contain only quiet threads. MAP already marks the link weak | MAP:81-95; EXP:3942 | Print the Redis cell as "DynSTC (quiet threads)" and keep the soundness text separate, as decision 5 of 12 Oct says |
| 26 | D | **MAP is dated 12 Oct and stale**: SQLite 1.19x by LO-OBJ (now 1.23x by LO-MARK v1); LO-MARK "Not adopted: other compiler root, premises not written up, no preservation run" (adopted 12 Oct, audited, preservation run); "Open: LO-OBJ with assertions, or LO-MARK"; FFmpeg 1.29x (AllOpt is 1.24x); Redis 1.54x (1.52x); MySQL 1.13x (1.10x); "two of about 26" Chromium runs (three of about 35) | MAP:3, 38-42, 127-131, 225-227, 273 | Update the six items |
| 27 | D | **LO-MARK guarantee wording in MAP is stronger than the audited wording.** MAP: "a checked access by a thread that does not hold the lock and meets the marker voids the run, so the ownership assumption is tested at run time"; terms table: "a wrong ownership guess is never a silently lost race". The supported wording adds "nor is ordered after its end", "under the stated premises", "apart from eviction", and that 8-9 of 16 premises are assumed | MAP:127-131, 282; OR:651-662 | Copy OR:651-662's wording |
| 28 | D | **MAP:5 "Hosts: AMD"** but the single figures it quotes for FFmpeg and Redis (DynSTC-RT +12.0, N1-ST +16.9, N1 +5.0, FE-INL +4.8, VWIDE +2.4) carry OR's Intel mark | MAP:5, 69, 73, 192, 220, 234; OR:189-195 | Mark them, or replace by the AMD singles of 13 Oct |
| 29 | D | **OR internal**: OR:94 "1.085x is the runtime option alone; the instrumentation gives 1.18x" against 1.138 and 1.13 (or 1.09 for fk6) in OR:71-72; OR:57 "FFmpeg (1) 1.18x"; OR:74 "compiler or runtime not separated" and PR:41 "a leg that separates compiler from runtime is queued" against the leg of 13 Oct (EXP:1134: the runtime, 4.2 %); OR:279 and PR:123 "MySQL 1.13x"; OR:283; OR:696 "SQLite's 1.24x"; OR:49 "Release" against `-O2 -g` RelWithDebInfo trees (PR:19, EXP:1133) | as listed | Update |
| 30 | D | **PR section 5 labels are inverted after 13 Oct**: "FFmpeg AllOpt with N1 at every check, +86 %, final" and "alternative: N1 placed statically, fk6, +18 %, current". AllOpt is fk6, whose compile time is from 2 builds in whole seconds | PR:118, 120; EXP:4007-4014 | Swap the labels; take fk6's compile time by the method (3 builds) |
| 31 | D | **PROTO:88 and PR:65-69 present Chromium as cumulative steps; they are not.** CR1 is the four analyses + FE-INL + MEMINTR without DE and peeling; the "+ DE" build has the library-facts table off while "the four analyses" has it on | PROTO:88; EXP:2514; `soundness-fixes.md`:518-519 | PR's row labels are already right; fix PROTO's list and add the table-off remark to the "+ DE" row |

## 2. Per-application matrix

Columns: mapped to (MAP) | in AllOpt | in which arm as built | correct by the Rule?

### SQLite (root 913c; AllOpt `z9L`, no run-time options)

| optimization | mapped to | in AllOpt | arm as built | correct? |
|---|---|---|---|---|
| STC | STC | yes | `s9t` (= stock) | yes |
| SWMR | SWMR | yes | `s9w` (= stock) | yes |
| SWMR-ROOTS | SWMR | no | `s9w` | extra, inert (identical to stock) |
| LO | LO | yes | `s9l` | yes |
| LO-MARK v1 (spec + site list) | LO (MAP says LO-OBJ, "not adopted") | yes | `s9l` | yes; MAP row stale |
| EA, EA-CONTENTS (default on) | EA | yes | `s9e` | yes |
| MEMINTR | EA | no | `s9e` | extra (finding 10) |
| DE, verified | DE | yes | `s9d` | yes; see finding 15 |
| peeling | DE | yes | `s9p` | yes; see finding 15 |
| N1 | DE | no | none | yes |
| DynSTC (compiled) | DynSTC | no | none | no arm |
| DynSTC-RT | DynSTC | no | options on stock | extra (finding 6) |
| FE-INL | own column | no | `s9f` | extra (finding 12) |
| library-facts table | none | default on | every arm | no mapping row |

Union of the columns contains AllOpt. Extras: SWMR-ROOTS, MEMINTR, DynSTC-RT, FE-INL.

### FFmpeg (root cr2; AllOpt `fk6`, beside it `fcn`; options `dynstc_rt=1,phase_guard=1,phase_guard_stats=1`)

| optimization | mapped to | in fk6 | in fcn | arm as built | correct? |
|---|---|---|---|---|---|
| STC, LO | STC, LO | yes | yes | `f2t`, `f2l` | yes |
| SWMR | SWMR | yes | yes | `f2w` | yes |
| SWMR-ROOTS | SWMR | no | no | `f2w` | extra (finding 11) |
| EA, EA-CONTENTS | EA | yes | yes | `f2e` | yes |
| MEMINTR | EA | no | no | `f2e` | extra (finding 10) |
| DE verified, peeling | DE | yes | yes | `f2d`, `f2p` | yes |
| N1 at every check | DE | no | yes | `f2d`, `f2p` | right for fcn, WRONG for fk6 (finding 1) |
| N1 placed statically (fk6's rule) | DE (unnamed rule) | yes | no | none | MISSING |
| N1-ST `=front` | DynSTC | yes | yes | none | MISSING (known; needs N1 and DynSTC-RT) |
| DynSTC-RT | DynSTC | yes | yes | options on stock | yes |
| `phase_guard=1` | (quiet threads' option) | yes | yes | options on stock | unnamed, no effect intended (finding 9) |
| FE-INL | own column | no | no | `f2f` | extra |

Union after the corrected DynSTC arm: equal to AllOpt only if the arm is built with fk6's placement; N1 is then in
two columns.

### Redis (root e52d; AllOpt `rdy`, whole-program; options `phase_guard=1,phase_guard_ranges=1,phase_guard_stats=1`)

| optimization | mapped to | in AllOpt | arm as built | correct? |
|---|---|---|---|---|
| summary step | none | yes | `r5T r5W r5L r5E`; not `r5d r5p` | no mapping row; finding 13 |
| STC, SWMR, EA (+EA-CONTENTS) | same | yes | `r5T`, `r5W`, `r5E` | yes |
| LO | LO | yes | `r5L` (= stock) | yes |
| SWMR-ROOTS, MEMINTR | SWMR, EA | no | none | yes |
| DE verified, peeling | DE | yes | `r5d`, `r5p` (unit by unit) | yes, but see findings 13, 14 |
| N1 at every check | DE | yes | `r5d`, `r5p` | yes, same placement as AllOpt |
| N1-ST `=fs` (carrier of the quiet bit) | DynSTC | yes | none | MISSING; not listed in MAP or OR |
| quiet threads = phase guard (six `-tsan-phase-*` flags, empty spec, options) | DynSTC (weak) | yes | none | MISSING (known) |
| DynSTC-RT | DynSTC | NO | options on stock | extra: not in AllOpt (finding 6) |
| FE-INL | own column | yes | `r5f` | yes |

Union after the corrected arm: equal to AllOpt in components. In gain it is not: N1 under quiet threads is worth about
+16 %, in no column unless the DynSTC arm contains N1.

### memcached (root ecc; AllOpt `kca`, whole-program, no run-time options)

| optimization | mapped to | in AllOpt | arm as built | correct? |
|---|---|---|---|---|
| summary step, `-tsan-summary-local-globals` | none | yes | `m4T m4W m4L m4E`; not `m4d m4p` | no mapping row; finding 13 |
| STC, LO | same | yes | `m4T`, `m4L` | yes |
| SWMR + SWMR-ROOTS | SWMR | yes | `m4W` | yes |
| EA, EA-CONTENTS | EA | yes | `m4E` | yes |
| EVCONF-CHECKED (spec, candidates) | EA (weak) | yes | none | MISSING (known) |
| EVCONF ranges, in-bounds, intercept, args | none ("not for the paper") | yes | none | MISSING; no mapping row |
| MEMINTR | EA | no | none (`m4e` superseded) | yes |
| DE verified, peeling | DE | yes | `m4d`, `m4p` | yes |
| N1 | DE | no | none | yes |
| DynSTC-RT | DynSTC | no | options on stock | extra |
| FE-INL | own column | no | `m4f` | extra |

Union after the corrected EA arm (`tsan-ea-wp` + the six EVCONF flags + the summary flags): equal to AllOpt.

### MySQL (root 8345; AllOpt `zyb`, no run-time options)

| optimization | mapped to | in AllOpt | arm as built | correct? |
|---|---|---|---|---|
| STC, LO | same | yes | `y8t`, `y8l` | yes |
| SWMR | SWMR | yes | `y8w` | yes |
| SWMR-ROOTS | SWMR | no | `y8w` | extra (finding 11) |
| EA, EA-CONTENTS | EA | yes | `y8e` | yes |
| MEMINTR | EA | no | `y8e` | extra (finding 10) |
| DE verified, peeling | DE | yes | `y8d`, `y8p` | yes |
| FE-INL | own column | yes | `y8f` | yes |
| DynSTC-RT | DynSTC | no | by the template | extra |

No single figure on record (finding 17).

### Chromium (root 8345; steps, not columns)

| component | mapped to | in CR1+N1 (the paper's AllOpt) | step that shows it | consistent with the applications? |
|---|---|---|---|---|
| STC, SWMR, LO, EA | same | yes | "the four analyses" 1.005 | yes |
| SWMR-ROOTS | SWMR | yes (inert) | hidden in the FE-INL + MEMINTR step | applications show it in the SWMR column; here it is in the FE-INL step. Harmless, but say "inert" in the row |
| MEMINTR | EA | yes | joint step with FE-INL, +9.8 %, share not separated | no: applications put it in the EA arm (three of five), Chromium in the FE-INL step |
| library-facts table | none | yes (default on) | CR1 without it reads the same | MAP says "in no adopted figure": false |
| DE | DE | NO | separate side step 1.012 (table off) | the paper's DE is not in the adopted configuration |
| peeling | DE | NO | side step 0.993 | not in the adopted configuration |
| FE-INL | own | yes | joint step | yes |
| N1 at every check | DE | yes | last step, +16.5 / +13.4 % | mapped to an analysis that is absent here (finding 19) |
| N1-ST, DynSTC-RT, quiet threads, EVCONF, LO-MARK | - | no | - | yes |

## 3. Flags and options of the adopted lines with no optimization name

`-tsan-summary-local-globals` (memcached); `-tsan-evconf-ranges-in-bounds` (memcached); `-tsan-phase-quiet-words`,
`-tsan-phase-hoist-front`, `-tsan-phase-config-table-proof`, `-tsan-phase-register-excluded` (Redis; parts of quiet
threads, undocumented in OR section 6); `phase_guard_ranges=1` (Redis; OR:612 mentions "range checks included");
`phase_guard=1` on FFmpeg (finding 9); `-tsan-summary-mismatch-fatal`, `-tsan-phase-program-digest`,
`phase_guard_stats=1` (safety and diagnostics, no effect on checks by their names; not verified in the source).
`-tsan-phase-guard-spec` on Redis points to an empty file (0 bytes), so no annotation enters. memcached's
`v4.owner.spec` states in its header that it is produced by the candidates run.

## 4. Can the arms be made consistent with the mapping?

Constraint of 13 Oct, applied here: the mapping stands; a change is proposed only as "X belongs under old name B
rather than A"; nothing becomes a column or an analysis of its own. Under that constraint no fix in section 1 changes
the mapping. What remains are three questions, because the Rule and the mapping together do not decide them.

The obstacle. N1-ST exists only inside N1's inline sequence and only with DynSTC-RT on (FFmpeg) or with the quiet bit
set (Redis). Quiet threads is obeyed through that same front test at inlined sites. N1 is under DE; N1-ST, DynSTC-RT
and quiet threads are under DynSTC. Read strictly ("only what is mapped to the column"):

- FFmpeg: N1-ST can be built in no column: it needs N1 (DE) and `dynstc_rt=1` (DynSTC) at once;
- Redis: quiet threads can be built under DynSTC without N1 (through the phase entries), but the column then shows
  about +20 %, not the +31.9 % of OR section 3 (which was measured with N1 present), and N1's gain under quiet
  threads (about +16 %) is in no column.

**Q1. What does a column contain when its optimization needs one mapped to another name?** This is a rule for
building arms, not a mapping row. Proposed reading: the arm contains the optimization and its prerequisite, and the
prerequisite is named under the table. Then DynSTC on FFmpeg = DynSTC-RT + N1 (AllOpt's placement) + N1-ST, and
DynSTC on Redis = quiet threads + N1 + N1-ST `=fs`, whole-program; DE on both stays verified DE + N1; the caption says
the two columns share N1 and are not additive. Without this reading the three corrected arms as requested cannot show
the recorded gains.

**Q2. N1-ST: DynSTC or DE?** Neither fits well. For DE: it is a change inside N1's sequence and cannot be compiled
without N1. For DynSTC: its condition is DynSTC's ("one thread"), it does nothing unless DynSTC-RT or the quiet bit is
on, and its whole gain appears only in single-threaded phases. Least bad: DynSTC, as now, because the property it
tests is DynSTC's and a move to DE would put a `dynstc_rt=1` run inside the DE column without helping Redis. No remap
proposed.

**Q3. The library-facts table** (on by default in every adopted line) feeds EA (argument-memory facts) and DE (the
"does not synchronise" fact); `soundness-fixes.md`:508-518 records both uses. It fits no single old name. Least bad:
EA, since the only code difference found on Chromium with the table off came through escape analysis (about 70 checks
of 12.8 million) and the DE use is the one with a known false row. Alternative that needs no mapping at all: pass
`-tsan-lib-facts=false` in the adopted lines (Chromium's table-off build reads the same) and keep it out of the paper.

Quiet threads under DynSTC (finding 25) and MEMINTR under EA (finding 23) stay where they are; both are weak links
already marked in MAP, and no other old name is better (quiet threads tests "no other thread is active", which is
STC/DynSTC's property and nothing else's; MEMINTR's non-trivial half is EA's fact).

The three corrected arms are new compile lines: each needs its own gate, void-free runs and a
preservation run before it is quoted (EXP:3946-3949, 3967-3969 record that quiet threads and EVCONF-CHECKED were
audited only inside the full lines).

memcached needs no mapping change: EVCONF-CHECKED + EA with the summary step is a subset of AllOpt. Whether its
elisions need SWMR-ROOTS or the DE of the full line is not established (section 5).

## 5. Not verified, and why

- The default of `-tsan-lib-facts` and of `-tsan-ea-contents-escape` on each root: inferred from EXP:2514, EXP:3896
  and `soundness-fixes.md`:505; the compiler source was not in the named directories.
- That root e52d has the phase entries for non-inlined sites: inferred from `rdn` reading 1.283 (EXP:1119), not from
  the source.
- Whether EVCONF-CHECKED's elisions fire in an EA-only whole-program line as in `kca` (dependence on SWMR-ROOTS, on
  the other analyses' facts): OR:200 measured EVCONF only together with EA-CONTENTS and SWMR-ROOTS; no record of it
  alone. The corrected arm's census line against `kca`'s will show it.
- Whether dominance elimination reads the whole-program summaries (finding 13).
- The run-time options of each timed leg were read from EXP's rows, not from the legs' environment records (on the
  measuring host, not contacted). MySQL's DynSTC arm and single-arm leg: no row yet.
- Offset trees (`..00`, `..16`, ...) were assumed to share their base tree's objects; only the `__` trees'
  build_info was read (plus `zys00`, `zyb00`).
- Chromium's build records (args files) were not read; its configurations are taken from EXP:2593-2595, 2514 and
  OR:676-683. Whether "Sound" and "Sound + DE + peeling" were built with the library-facts table on: not found.
- The byte identity of "= stock" arms is the READY lists' statement, not rechecked.
- Figures in PR sections 2, 3, 5 were compared with OR and EXP rows and agree except where listed (findings 2, 3,
  29, 30); the raw result files were not read.
