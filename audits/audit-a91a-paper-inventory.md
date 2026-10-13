# Audit A91a: inventory of every stale item in the paper against the measured results

Date: 14 Oct 2026. Read-only. Object: `/home/alexey/tsan-instr-paper` at `origin/main` (78b6ceb; the working copy
adds one plan commit and no paper change) and the three open branches `origin/fe-inl`, `origin/redis-results`,
`origin/methodology`. Compared against `paper-results.md` (last change 14 Oct 00:24; "PR" below, with its section
number), `optimization-results.md` ("OR"), `measurement-protocol.md`, `camera-ready-mapping.md`, and audit A90.
Nothing was built or run; page positions are taken from the plan file and from the stale `main-atc.aux`.

Verdicts: READY (final value, number or plot swap only) · READY-WORDING (final value, sentence must change) ·
PROVISIONAL (value exists, a re-read is queued or its status is "current") · MISSING-DATA · DECISION ·
DROP-CANDIDATE.

## 0. Result in one screen

Marked items on `origin/main`: 58 `\stale`, 41 `\staleclaim`, 9 `\stalefig` floats = 108.

| verdict | `\stale` | `\staleclaim` | `\stalefig` | total |
|---|---|---|---|---|
| READY | 19 | 0 | 0 | 19 |
| READY-WORDING | 10 | 26 | 0 | 36 |
| PROVISIONAL | 10 | 0 | 2 | 12 |
| MISSING-DATA | 12 | 0 | 2 | 14 |
| DECISION | 3 | 11 | 3 | 17 |
| DROP-CANDIDATE | 4 | 4 | 2 | 10 |
| total | 58 | 41 | 9 | 108 |

On `origin/redis-results` five further marks exist (3 `\prov` in text, one `\prov` tick label, one
`\provfigstyle` panel); all five are READY now (block B4).

Three findings that cut across the blocks:

- **X1. "final" in PR does not always mean "in the decided DE form".** The decision is removal-form DE for every
  application but MySQL. Record cells in removal form exist for Redis and memcached only. FFmpeg's record cell
  (1.27×) is stated in PR 2 as "DE in its verified form" and OR says "the record cell with removal follows";
  SQLite's record leg (1.23×) predates the decision and the screening row names its adopted line as the verified
  form. Both have a 2 offsets × N = 2 screening that reads the same (FFmpeg 1.273 against 1.271; SQLite 1.27
  against 1.27), so the figures are safe to enter, but as `\prov`, not as final. memcached's DE and DE+Peeling
  columns are still the verified form (PR 2: "removal re-read is queued"). For the DE and DE+Peeling columns of
  FFmpeg and SQLite the form of the timed trees is not stated in PR; not verified here.
- **X2. Every figure in PR 2 is over the compiler root's own stock, and the base is an open decision.** Over
  upstream stock: memcached 1.82× (not 1.91×), Redis 1.43× (1.47×), FFmpeg 1.26× (1.27×), SQLite 1.25× (current,
  not resolved), MySQL unchanged. The single-configuration columns exist over the root's stock only. All READY
  verdicts below assume the root's stock; choosing upstream stock changes three headline numbers and leaves the
  single columns without a measured base.
- **X3. The direction of four claims has reversed**, so their blocks cannot be number swaps: DE is no longer the
  strongest configuration anywhere (B3-B8, B16); memory overhead rises in four of five applications (B12);
  instrumentation counts rise, not fall (B11); the gain falls as the thread count rises on memcached (B10).

## 1. Inventory by block

Each block is one candidate branch. "Line" = line in `origin/main`.

### B1. Methodology paragraph (`4-experiments.tex:11-20`)

| line | now | needs | replacement (PR) | verdict |
|---|---|---|---|---|
| :11 `\staleclaim` | six applications with `redis-benchmark`, `sysbench`, `threadtest3`; "I/O-bound servers to CPU-intensive desktop applications" | workload per application | PR 1, final: SQLite threadtest3 stress2 (60 s); FFmpeg four transcodes of one film; Redis 7 data-heavy commands, 8 I/O threads; MySQL sysbench insert/update/delete, 24 connections, `-O2 -g`; memcached pipelined 32-key gets, 190-byte keys; Chromium 127.0.6533.119, component build with DCHECKs, blink_perf (two story samples of 49 and 48 stories) and Speedometer 3.0. PR line 9-10: the text must say that the workloads differ from the submission's | READY-WORDING |
| :13-15 `\staleclaim` | Intel host, Ubuntu 22.04, Clang 19, "run multiple times" | hosts, N, statistics | replaced by `\upd` on `origin/methodology` (AMD 2 × EPYC 9115, Chromium on Intel, 4 builds × 4 runs for stock and AllOpt, 2 × 2 for the others, native one build and 16 runs, ratios of geometric means, 95 % bootstrap). Consistent with the protocol. Not covered by that text: Chromium is N = 3 without code offsets; Redis DE and DE+Peeling are 2 offsets × N = 4. "Ubuntu 24.04" and "LLVM/Clang 19" are not in PR: not verified | READY-WORDING (text exists on the branch) |
| :16 `\staleclaim` | AllOpt = EA, LO, STC, SWMR, DE | definition of AllOpt | PR 2: AllOpt is "the adopted configuration" per application; FE-INL is part of it only for Redis, MySQL, Chromium; the run-time forms (ownership checks, lock marks, quiet threads, inline test) are in it. The wording depends on plan item 11 | DECISION (how the run-time forms of EA and LO are described; AllOpt per application) |
| :17 `\upd` | "two variants ..., and TSan+FE-INL" | — | `origin/fe-inl` rewrites it and adds the defining paragraph; on `main` alone the name FE-INL is used before it is defined | merge `fe-inl` |
| :20 unmarked | "we vary the number of active threads for a subset of benchmarks" | sweep | true for memcached and FFmpeg (PR 8); see B10 | keep |

Merge note: `methodology` (lines 13-16) and `fe-inl` (line 17) change adjacent lines of the same hunk; expect a
textual conflict on the second merge.

### B2. Baseline overhead (`4-experiments.tex:78-80`, `1-introduction.tex:45`, `4-plots-original-slowdown.tex`)

Quantity: stock TSan over native (PR 3), and AllOpt over native for the figure.

| line | now | replacement | status | verdict |
|---|---|---|---|---|
| exp:78 range low | 2.4× | 2.8× (FFmpeg 2.76×) | final | READY |
| exp:78 range high | "over 19×" | needs Chromium native; without it the top is MySQL 10.0× | — | MISSING-DATA |
| exp:78 SQLite | 2.4× | 5.43× [5.12, 5.68] | final | READY |
| exp:78 memcached | 2.5× | 8.48× [8.39, 8.58] (upstream stock 8.09×) | final | READY |
| exp:78 `\staleclaim` | "Disk- and network-heavy ... moderate slowdowns as I/O latency masks" | contradicted: memcached is second highest | — | READY-WORDING |
| exp:80 FFmpeg | 2.9× | 2.76× | final | READY |
| exp:80 Redis | 9.2× | 6.21× (`\prov{6.2}` on `redis-results` is now final) | final | READY |
| exp:80 MySQL | 7.1× | 10.03× [9.94, 10.15] | final | READY |
| exp:80 `\staleclaim` | "In contrast, in-memory databases ... substantially higher" | new order: MySQL 10.0, memcached 8.5, Redis 6.2, SQLite 5.4, FFmpeg 2.8 | — | READY-WORDING |
| exp:80, intro:45 (2 × 3 numbers) | Speedometer 3.65 → 0.19, 19.2× | no native Chromium build exists (PR 3, PR 5: "pending (native build)") | — | MISSING-DATA × 6 |
| Fig. 7 `\stalefig` | 12 bars | stock / AllOpt over native: Redis 6.2 / 4.2; MySQL 10.0 / 9.1; FFmpeg 2.8 / 2.2; memcached 8.5 / 4.4; SQLite 5.4 / 4.4 (all final; FFmpeg and SQLite AllOpt in the verified form, X1). Chromium pair: none. `ymax=12` still fits | 10 of 12 bars final | MISSING-DATA (Chromium panel only) |

Unmarked in this block: "Speedometer~3.1" at exp:80 and intro:45 (the tree has 3.0 only, PR 1); "Chromium is the
extreme case" (unknown until native exists); legend "Best optimized result" and caption "the best result
achieved by our optimizations" (`4-plots-original-slowdown.tex:79,112`; the new bars are AllOpt, and SQLite's
best single configuration is above AllOpt); on `redis-results` the AllOpt bar reads 4.1, the removal-form cell
gives 4.22.

### B3. memcached (`4-experiments.tex:91`, Fig. 8a)

| item | now | replacement | status | verdict |
|---|---|---|---|---|
| `\staleclaim` + AllOpt | "highly optimized, network-bound ... modest speedup of 1.07x" | 1.91× [1.90, 1.92], removal form (1.82× over upstream stock); now the largest gain | final | number READY; claim READY-WORDING |
| `\staleclaim` + DE, DE+Peeling | "strongest contributors are DE and DE+Peeling, each 1.03x" | DE 0.98 [0.96, 1.00], DE+Peeling 1.00, both verified form; the carrier is EA with its run-time checked form, 1.86 [1.84, 1.87] | removal re-read queued | number PROVISIONAL; claim READY-WORDING |
| panel bars (EA, LO, STC, SWMR, DE, DE+Peeling, DynSTC, FE-INL \| AllOpt) | 1.00 1.01 1.00 1.00 1.03 1.03 0.98 — \| 1.07 | 1.86 1.00 1.00 1.01 0.98p 1.00p 1.00 1.00 \| 1.91 | final but the two p | see B8 |

### B4. Redis (`4-experiments.tex:94`, Fig. 8b; partly done on `origin/redis-results`)

| item | on `main` | on the branch | replacement now | verdict |
|---|---|---|---|---|
| AllOpt | 1.45x | `\prov{1.52x}` twice | 1.47× [1.45, 1.49], removal form, record cell 13 Oct evening (1.43× over upstream stock) | READY |
| `\staleclaim` DE strongest 1.35x | — | `\upd`: "No single configuration exceeds 1.14x (DynSTC); ... only by combining DynSTC, the inline test and FE-INL" | still true with DE 1.11 | READY (branch text holds; change 1.52 → 1.47) |
| panel | 7 + 1 bars | 1.01 1.00 1.00 0.99 1.06 1.07 1.14 1.07 \| 1.52, `\provfigstyle` | DE 1.11 [1.08, 1.13], DE+Peeling 1.08 [1.06, 1.10], AllOpt 1.47; others unchanged | READY |
| slowdown | 9.2 / 6.3 | 6.2 / 4.1, tick `\prov{Redis}` | 6.21 / 4.22 | READY |

All five `\prov` marks of the branch and its `\provnote` can be removed with these values. Unmarked: "benefits
significantly", "average speedup" (a composite of seven commands); DynSTC 1.14 comes from a 2 × 2 leg whose A/A
reads 0.985 [0.962, 1.007].

### B5. FFmpeg (`4-experiments.tex:97`, Fig. 8c)

| item | now | replacement | status | verdict |
|---|---|---|---|---|
| AllOpt | 1.57x "average" | 1.27× [1.27, 1.28] (static placement, `dynstc_rt=1` only; 1.26× over upstream stock) | final in PR, verified form; removal record cell to follow (X1) | PROVISIONAL |
| DE | 1.30x | 1.02 [1.00, 1.03] | final; form not stated | PROVISIONAL |
| DE+Peeling | 1.42x | 1.01 [1.00, 1.02] | final; form not stated | PROVISIONAL |
| `\staleclaim` | "DE is the strongest single optimization ..., the full combination yields the best" | DynSTC 1.28 [1.26, 1.30] is the carrier and reads above AllOpt (1.27); per transcode the gain is the stream copy (2.53×), mjpeg 1.03, h265 1.00, h264 1.01 | — | READY-WORDING |
| panel | 1.05 1.00 1.00 1.00 1.30 1.42 1.15 — \| 1.57 | 1.00 1.00 1.00 1.00 1.02 1.01 1.28 1.00 \| 1.27 | | see B8 |

Unmarked: "This CPU-intensive multimedia framework also shows substantial speedups" (three of four transcodes
are at 1.00-1.03).

### B6. MySQL (`4-experiments.tex:100`, Fig. 9a-b)

| item | now | replacement | status | verdict |
|---|---|---|---|---|
| `\staleclaim` | "complex I/O-bound DBMS, improvements more modest but consistent" | 1.10×, carried by FE-INL alone | — | READY-WORDING |
| Select: AllOpt 1.16x, DE 1.15x; Fig. 9a | select-random-points | no select workload is measured | — | DROP-CANDIDATE × 2 (else MISSING-DATA: a select leg) |
| Write: AllOpt 1.11x | `write-only` | 1.10× [1.09, 1.11] over insert/update/delete (delete 1.14, insert 1.09, update 1.07); verified form by decision | final | READY (workload name changes) |
| Write: DE 1.11x | | DE 1.00 [0.99, 1.00]; FE-INL 1.09 [1.08, 1.09] | final | READY |
| `\staleclaim` (sentence carrying the four numbers) | "DE remains the most effective analysis" | contradicted | — | READY-WORDING |
| `\staleclaim` | "smaller gains reflect that TSan's overhead is a smaller share of total time" | contradicted: MySQL has the highest stock slowdown (10.0×) | — | DROP-CANDIDATE |
| panel b | 1.04 1.02 1.04 1.03 1.11 1.08 1.01 — \| 1.11 | 1.01 1.00 1.00 1.00 1.00 0.99 1.00 1.09 \| 1.10 | final | see B8 |

### B7. SQLite (`4-experiments.tex:103-104`, Fig. 8d)

| item | now | replacement | status | verdict |
|---|---|---|---|---|
| AllOpt | 1.71x, "highest application-level speedup" | 1.23× [1.20, 1.27] (1.28× with the other half of the host idle); no longer the highest | final in PR; verified form, removal screening only (X1) | PROVISIONAL |
| best single | 1.24x | LO 1.26× [1.22, 1.31], above AllOpt | final | READY |
| `\staleclaim` | "the full combination matters most ... clear synergistic effect" | contradicted. PR 4 adds that the figure counts "statements OK" dominated by trivial statements, that successful data-modifying statements fall (0.75-0.88×), and that the gain disappears under a perf attachment or below about four CPUs; "the paper must name the metric" | — | DECISION (metric named: statements OK or attempted; how much of the caveat is stated) |
| panel | 1.17 1.01 1.00 1.00 1.21 1.24 1.00 — \| 1.71 | 0.97 1.26 1.00 1.00 1.01 1.00 1.00 1.02 \| 1.23 (STC, SWMR byte-identical to stock; all but LO inside the A/A of 1.014 [0.938, 1.073]) | | see B8 |

Not resolved here: an earlier decision note of 12 Oct reads "SQLite: no DE, no peeling", while PR 2 says
"SQLite's adopted line is built with DE and peeling". Which one the record figure represents must be stated
before the number goes in.

### B8. Speedup figures (`4-plots-speedup.tex`)

| float | content | replacement | verdict |
|---|---|---|---|
| Fig. 8 (`:6-104`), four panels | bars above (B3, B4, B5, B7) | all bars exist; needs a ninth tick (FE-INL) in `allbenchstyle` (`:16-17`), new `ymax` (memcached 2.0), caption "Average Speedup" | PROVISIONAL (memcached DE, DE+Peeling; FFmpeg and SQLite removal form); fillable today with those bars under `\provfigstyle` |
| Fig. 9 (`:111-225`), MySQL select, MySQL write, Chromium Parser, Chromium Image Decoder | per-configuration bars | MySQL write: B6 (final). MySQL select: none. Chromium per benchmark and per single configuration: none; Chromium was measured cumulatively only (four analyses 1.005; + DE 1.012; + peeling 0.993; + FE-INL + MEMINTR 1.104 / 1.101; AllOpt 1.286 / 1.248) | DECISION (new design; the natural one is MySQL as a fifth panel of Fig. 8 and one cumulative Chromium panel) |
| Fig. 10 (`:231-316`), box plot per Chromium suite; caption 1.39×, 1.14×, 2.16× | per-suite distributions | PR has sample-level figures only; 1.39× → 1.286 (first sample) / 1.248 (hold-out); 1.14× → Speedometer Score 1.225 (Intel, N = 2 of 3), not a median; 2.16× → none | float DROP-CANDIDATE; caption numbers READY-WORDING × 2, MISSING-DATA × 1 |

Unmarked: `\Description` at `:222` (1.16x, 1.11x, 1.32x, 1.62x); sub-captions `select-random-points`,
`write-only` (`:150,172`); tick labels without FE-INL (`:17,126`). Stated in PR 2 and to be carried into the
caption or text: the DynSTC column of SQLite, memcached and MySQL is the run-time option, which their adopted
lines do not use; the EA and SWMR columns of SQLite, FFmpeg and MySQL include MEMINTR and SWMR-ROOTS.

### B9. Chromium text (`4-experiments.tex:107-109`)

| item | now | replacement | status | verdict |
|---|---|---|---|---|
| `\staleclaim` (whole sentence group) | "suite including Speedometer 3.1 and Blink benchmarks. Across all tests ..." | two story samples of blink_perf (49 + 48 stories, six suites) and Speedometer 3.0, Intel host, N = 3 | final | DECISION (which figure leads: first sample 1.29×, hold-out 1.25×, or Speedometer 1.23×; whether the AMD figures 1.08 / 1.02 / 1.006 are stated) |
| geomean | 1.39x | 1.286 / 1.248; whether these are geometric means over stories is not stated in PR | final | READY-WORDING |
| Speedometer median | 1.14x | Score ratio 1.225 (1.201, 1.250; N = 2, one run died on a V8 debug check) | final | READY-WORDING |
| Image Decoder 2.16x, Layout 1.45x | suite medians | none in PR | — | MISSING-DATA × 2 (a per-suite tabulation of the existing per-story data, or drop the clause) |
| `:109` `\staleclaim` | "This confirms our approach is effective on large, real-world applications." | the paper's analyses read 0.99-1.01 on Chromium; the gain is FE-INL + MEMINTR and the inline test | — | READY-WORDING |

Unstated in the paper and material: 3 of about 46 Speedometer runs of AllOpt died on a V8 debug check (stock
dies on it only under forced compaction); AllOpt's peak memory is 1.54× stock and its code 3.0× stock.

### B10. Thread count (`4-experiments.tex:115-116`, `appendix/contention.tex:2`, `4-plots-contention.tex`, the "peak" in abstract, introduction, conclusion)

| item | now | replacement (PR 8) | status | verdict |
|---|---|---|---|---|
| exp:115-116 `\staleclaim` | "Memcached and SQLite ... optimizations, especially DE, remain effective as the thread count increases" | memcached AllOpt over stock at 8 / 6 / 4 / 2 server threads: 1.90 / 1.97 / 2.11 / 2.23: the gain falls as threads rise. FFmpeg composite at 16 / 8 / 4 / 2: 1.25 / 1.26 / 1.28 / 1.29: flat. SQLite: 1.00 at every count on walthread1, which never reaches the covered objects; "SQLite's gain has no thread-count series" | final, except memcached at 2 threads (current) | READY-WORDING |
| appendix `\staleclaim` | DE dominant in SQLite, AllOpt tracks DE | no per-configuration series exists; only stock and AllOpt were swept | — | READY-WORDING (rewrite to three sentences) |
| Fig. (appendix) `\stalefig` | 8 series × 20 thread counts, three panels, FFmpeg per transcode | 2 series × 4 thread counts; FFmpeg per transcode: only h264 and the stream copy's range (2.4-2.6×) are in PR | partly final | DECISION (redesign as a small table; SQLite panel has no content) |
| abstract:25, intro:150, conclusion:15 | "peak ... 2.54× under high thread contention" | 2.11× (4 threads, final) or 2.23× (2 threads, current); or the FFmpeg stream copy 2.53× (final, not a thread effect). "under high thread contention" is the opposite of what was measured | — | READY-WORDING × 3 (with a choice of number; dropping the clause is the smallest change) |

### B11. Instrumentation reduction (`4-experiments.tex:127-138`, Fig. 11a-c, abstract:24-25)

| item | now | replacement | status | verdict |
|---|---|---|---|---|
| exp:127 `\staleclaim` | "Runtime performance gains result from our analyses pruning instrumentation." | PR 4: the count of checks is above stock in every application (+5.3 % to +27.0 %); the gain is in checks that no longer enter the runtime | — | DECISION (how reduction is presented; open in the plan) |
| exp:132 SIR 64.9 % (FFmpeg), 60.8 % (memcached) + claim | "highly effective", "prove large portions of code to be race-free" | removed by the paper's analyses, pass statistics: FFmpeg 2.7 %, memcached 12.4 %, Redis 7.1 %, SQLite 1.4 %, MySQL 0.6 % | final | DECISION × 3 (two numbers and the claim; values are final, the metric is not chosen) |
| abstract:24 claim + 65 % | "a significant fraction ... statically eliminated, with up to 65 %" | at most 12.4 % removed; or "runtime calls avoided" up to 91.6 % (Redis, current), a different quantity | — | DECISION × 2 |
| exp:134 DIR 51.9 % (FFmpeg), 50.0 % (SQLite) | executed checks removed | drop in runtime entries: FFmpeg about 78 / 23 / 5 / 3 % per transcode, SQLite 11.8 % on stress2; memcached 76.9 %, Redis 94.5 %. This is entries into the runtime, not the paper's DIR (checks executed in any form), and one trace window per test | current | PROVISIONAL × 2 |
| exp:134 `\staleclaim` | "The static reduction translates into a substantial decrease ... explaining the observed performance gains" | not a consequence of static reduction any more; for SQLite a 12 % drop "does not by itself account for 1.23-1.28×" | — | DECISION |
| abstract:25 claim + "23 and 58 %" | DIR range | 11.8 % to 94.5 % as runtime-entry drop over four applications (current). MySQL on the record workload (insert/update/delete): not measured; PR 4 gives other sysbench scripts (0.7-16.6 %) | current | DECISION (claim), PROVISIONAL (number) |
| exp:136-138 (3 claims, 43.9 %, 41.3 %), Fig. 11c | DMIR | not re-measured by decision; the figures belong to the submitted workloads and configurations | — | DROP-CANDIDATE × 5 |
| Fig. 11 `\stalefig` | four panels | (a) per decision above, Chromium bar: none; (b) four applications current, MySQL none, different metric; (c) none; (d) see B12 | — | DECISION |

Unmarked: the footnote at exp:127-129 ("DIR and DMIR not reported for Chromium"); axis labels "Sites Removed",
"Checks Removed" and the caption "Instrumentation Reduction ... (higher is better)"; the metric definitions SIR,
DIR, DMIR in Fig. 6 (exp:54-60), which do not match what was counted; `appendix/rex-details.tex:4-5` compares
the oracle (submitted workloads) with `(1-DIR)` of AllOpt.

### B12. Memory (`4-experiments.tex:143-148`, Fig. 11d)

| item | now | replacement (PR 3) | status | verdict |
|---|---|---|---|---|
| exp:143 `\staleclaim` | "a reduction in memory overhead for almost all programs" | a reduction for memcached only (10 %); increases elsewhere | final | READY-WORDING |
| FFmpeg 5.5 % | | −0.3 % (static line; 1439 → 1442 MiB) | final | READY |
| MySQL 6.0 % | | −0.7 % | final | READY |
| SQLite 4.4 % | | −1.7 % | final | READY |
| exp:148 `\staleclaim` | "mainly driven by Escape Analysis ... Redis neutral" | Redis −0.5 %; no per-analysis memory was measured | final | READY-WORDING (drop the attribution) |
| Fig. 11d | six bars, axis 0-10 % | five values, four negative; Chromium: no overhead reduction without native, only peak memory 1.54× stock (Intel, current) | | part of Fig. 11 |

Unmarked and now unsupported: exp:145-146 (lazy shadow allocation as the reason for a reduction) and
`6-related-works.tex:91` ("eliminating instrumentation reduces shadow memory indirectly").

### B13. Compile time (`4-experiments.tex:156-158`, Fig. 12, abstract:26-27, intro:150, conclusion:16)

Quantity: (T_AllOpt − T_native) / T_native (PR 5, first table).

| item | now | replacement | status | verdict |
|---|---|---|---|---|
| exp:157 FFmpeg | 2.5 % | +38 % (static line; +108 % with the inline test at every check) | final | READY |
| exp:158 maximum | 18 % | +189 % (Redis) | current (across windows) | PROVISIONAL |
| abstract:27 range | 1-18 % | +25 % (MySQL, current) to +189 % (Redis, current); final ones: FFmpeg +38 %, memcached +138 % | mixed | PROVISIONAL |
| exp:156, exp:158, abstract:26, intro:150, conclusion:16 `\staleclaim` × 5 | "modest", "moderate", "low enough for integration" | not tenable at +25 % to +189 %; SQLite adds 98.7 s of spec generation per source version; memcached's whole-program step is 6.5 s of 16.3 s | — | READY-WORDING × 5 |
| Fig. 12 `\stalefig` | six bars, axis to 23 % | memcached +138 (final), FFmpeg +38 (final), Redis +189, SQLite +69, MySQL +25 (current); Chromium: none over native (+10 % CPU time over stock on 400 sampled units, final) | mixed | PROVISIONAL |

Unmarked: "For a large project like FFmpeg, ... only" (exp:156-157); abstract:10 "to ensure low compilation
overhead"; conclusion:16-17 "underscoring the practicality"; `\label{fig:cto_overhead}` sits outside a caption
(`4-plots-overheads.tex:40`, pre-existing).

### B14. Race detection (`4-experiments.tex:162-166`)

| item | now | replacement (PR 6, PR 7) | status | verdict |
|---|---|---|---|---|
| exp:162 `\staleclaim` | "TSan+AllOpt detected all races identified by Original TSan" | per application: no site lost and no seed below stock for the removal form on all five; SQLite singles and memcached singles final; SQLite adopted line, Redis, FFmpeg, MySQL, Chromium "current"; three rare SQLite sites and 3-14 MySQL sites per arm "undetermined"; Chromium PartitionAlloc race 11 of 21 (stock) against 2 of 19 (AllOpt), unexplained; per-site rates on the final binaries pending | mixed | READY-WORDING |
| exp:166 `\staleclaim` | "This confirms that our optimizations preserve TSan's race detection." | as above; the regression suite (PR 7) has failing rows for the adopted lines of FFmpeg (12) and Redis (2), none a missing report | mixed | READY-WORDING |

This is also where the requested preservation protocol text goes (plan S2); it adds text, see section 5.

### B15. Headline numbers and claims (abstract, introduction, conclusion)

| item (3 places unless noted) | now | replacement | verdict |
|---|---|---|---|
| "up to 1.71×" | SQLite | memcached 1.91× (1.82× over upstream stock) | READY × 3 (after X2) |
| "geometric-mean Chromium speedup of 1.39×" | all tests | 1.25× (hold-out) / 1.29× (first sample); Intel only | READY-WORDING × 3 |
| "peaks of 2.54×" | see B10 | | READY-WORDING × 3 |
| compile-time claims | see B13 | | counted there |
| abstract: 65 %, 23-58 % and their claims | see B11 | | counted there |
| abstract:27, intro:149 `\staleclaim` "fully automatic" | | SQLite's line needs the names of the mutex API and object types and a generated spec; memcached's a candidate list from a separate tool; Redis's phase flags and a base spec | DECISION × 2 (plan item 11) |
| intro:74-75 `\staleclaim` "without introducing any additional runtime filtering logic" | | the adopted lines carry inline tests, ownership checks, lock marks, a phase guard | DECISION (plan item 11) |

### B16. Summary box (`4-experiments.tex:190`)

Both `\staleclaim` spans ("AllOpt yields the highest gains, confirming a synergistic effect"; "DE is
consistently the strongest single optimization") are contradicted: SQLite's LO is above AllOpt, FFmpeg's DynSTC
equals it, DE is 0.98-1.11 everywhere. READY-WORDING × 2. The unmarked last sentence ("Other analyses eliminate
instrumentation where DE is inapplicable") falls with them. The overview sentences at exp:86-88 (two
`\staleclaim`) are the same claims: READY-WORDING × 2.

### B17. Chromium appendix (`appendix/chromium-selected.tex`)

| float | now | replacement | verdict |
|---|---|---|---|
| Fig. (`:9-90`) Speedometer, Paint, SVG per configuration | 8 bars each | Speedometer: two configurations only (1.112, 1.225); Paint suite on the AMD host only; SVG: none | DROP-CANDIDATE |
| Fig. (`:92-153`) CDF over all tests; caption median 1.32×, about 93 % at or above 1.0× | | derivable only from the per-story data of the 97 stories; not in PR | MISSING-DATA × 3 (float and two numbers; a tabulation, not a new run), or drop with Fig. 10 |

Unmarked: "Speedometer~3.1" at `:4`, `:50`, `:88`.

## 2. Unmarked stale items

Still unmarked after A90's list was applied (A90 numbers in brackets where it had them):

| where | text | against |
|---|---|---|
| `abstract.tex:10`; `1-introduction.tex:69-70`; `7-conclusion.tex:6-7` [D1] | "never cause TSan to miss races it would otherwise detect"; "with a guarantee that no races are missed"; "without compromising soundness or completeness" | the run-time checked forms declare a run void on a violation and produce no report for that access (plan item 11); eviction-only losses are accepted; removal-form DE with post-dominance adds premises (plan, open decision) |
| `abstract.tex:28` | "places no additional burden on developers" | same ground as "fully automatic" |
| `abstract.tex:7`; `1-introduction.tex:88-89` [D9] | "a significant fraction of which is for safe accesses" | the removing analyses take 0.6-12.4 % of the checks |
| `abstract.tex:24` | application list omits FFmpeg | pre-existing |
| `abstract.tex:25`; `7-conclusion.tex:15` [D10] | "under high thread contention"; "in contention-heavy workloads"; "significantly reduces" | B10 |
| `1-introduction.tex:45`; `4-experiments.tex:80,107`; `appendix/chromium-selected.tex:4,50,88`; `4-plots-original-slowdown.tex:112` [D11] | "Speedometer 3.1" | the measured benchmark is Speedometer 3.0 (PR 1) |
| `1-introduction.tex:72,146`; `7-conclusion.tex:10-13`; `3-approach.tex:137` [D15] | "five complementary static analyses" as the system | the gains come from the extensions and FE-INL; text matter |
| `1-introduction.tex:76,143`; `3bb-transformations.tex:101-103` [D5, D6] | "per-access cost of one atomic read and branch"; "experiments show it is small"; "almost-zero overhead" | the run-time option alone costs the h264 transcode about 7 % on the stock binary (PR 2 notes); 1.00 in the record cell |
| `3c-pipeline.tex:208-209` [D7]; `6-related-works.tex:73` [D8] | "operate per compilation unit, avoiding LTO"; "without requiring programmer directives" | memcached and Redis use the whole-program summary step (described in the same paragraph, so consistent, but its cost is now +75 % to +168 % of the build); SQLite, FFmpeg, MySQL are built unit by unit, "where STC, SWMR and LO remove nothing" |
| `4-experiments.tex:94,97` [D3] | "benefits significantly", "substantial speedups", "average" | B4, B5 |
| `4-experiments.tex:145-146`; `6-related-works.tex:91` | fewer instrumented accesses lower memory | B12 |
| `4-experiments.tex:25-67` [D13] | definitions of SIR, DIR, DMIR, CTO | B11; CTO over native now has data |
| `4-experiments.tex:182`; `appendix/rex-details.tex:4-7` [D14] | ReX and oracle figures | kept by decision; the text should say they are from the submitted workloads |
| `4-plots-speedup.tex:222` [M21] | `\Description` with four stale figures | invisible in the PDF; cannot carry a colour mark |
| `4-plots-speedup.tex:150,172`; `:17,126` [M24] | sub-captions of the MySQL panels; tick labels without FE-INL | B6, B8 |
| `4-plots-original-slowdown.tex:79,112` [M22] | "Best optimized result" | B2 |
| methodology text on `origin/methodology` | "Ubuntu 24.04 LTS", "LLVM/Clang 19" | not in PR; not verified. Thread counts (Redis 8 I/O threads, MySQL 24 connections, FFmpeg 4 threads, memcached 8 server threads) and the Chromium version and build type are nowhere in the paper |
| nowhere in the paper | the base of the speedups (X2); that the AMD record legs ran with the host's other half busy; SQLite's metric | PR 2, PR 4 |

## 3. Answers

### (1) Fillable today with numbers and plots only

- B4 Redis: everything, by updating `redis-results` (1.47, DE 1.11, DE+Peeling 1.08, slowdown 6.2 / 4.2) and
  removing its provisional marks.
- B2: the five application numbers at exp:78-80 and ten bars of Fig. 7 (the two sentences around them need
  wording; the Chromium part stays red).
- B6 MySQL: 1.10 and the single-panel bars.
- B3 memcached: 1.91 and seven of nine bars final; DE and DE+Peeling as `\prov`.
- B5 FFmpeg, B7 SQLite: all bars and the AllOpt figures as `\prov` (X1); LO 1.26 final.
- B12: the three memory numbers (the sentence around them reverses).
- B13: FFmpeg +38 % final; Fig. 12 bars for memcached and FFmpeg final, the other three as `\prov`.
- B10: the memcached and FFmpeg sweep values (as a table).

Number-only verdicts: 19 READY, plus 12 PROVISIONAL that can be entered under `\prov`.

### (2) Blocked only by a measurement: ranked list

| rank | missing measurement | unblocks |
|---|---|---|
| 1 | Chromium native build, run on Speedometer 3.0 and the first story sample on the Intel host (score, peak memory), and its build time on the 400-unit sample | 8 marked items: intro:45 (3), exp:78 (1), exp:80 (3), Fig. 7 (its Chromium pair); plus the Chromium bars of the memory and compile-time panels |
| 2 | Removal-form re-reads: memcached DE and DE+Peeling (queued); FFmpeg record cell and its DE, DE+Peeling columns; SQLite record cell and its DE, DE+Peeling columns | turns 7 PROVISIONAL items and Fig. 8 final: exp:91 (1), exp:97 (3), exp:103 (1), and the AllOpt bars of Fig. 7 for FFmpeg and SQLite |
| 3 | Chromium per-story tabulation of the existing runs (per-suite medians, distribution, share at or above 1.0, geometric mean); no new run if the per-story records exist | 7 items: exp:107 (2), Fig. 10 and one caption number, appendix CDF and its two numbers. Alternative: drop all of them |
| 4 | Compile time over native in one window, 3 builds per arm, for Redis, SQLite, MySQL | 3 PROVISIONAL items: exp:158, abstract:27, Fig. 12 |
| 5 | Dynamic count per the paper's definition (checks executed in any form, and entries into the runtime beside it), final status, for the five applications; includes MySQL on its record workload, which has no figure at all | exp:134 (2), abstract:25 (1), Fig. 11b; also needed before the decision on B11 can be closed |
| 6 | memcached sweep point at 2 threads at N = 2 on a quiet host; FFmpeg per-transcode series tabulated from the sweep's records | the 2.23× candidate for the "peak" (3 places), the contention figure |
| 7 | Chromium static count (pass statistics) | one bar of Fig. 11a |
| 8 | Per-site preservation rates on the final binaries (PR 6: pending) | lets B14 be worded as final |
| — | not recommended to measure: a MySQL select workload (Fig. 9a, 2 numbers); Chromium single configurations per benchmark (5 panels); a SQLite thread series (no workload with a variable thread count reaches the covered objects) | redesign or drop instead |

### (3) Blocks that need wording, and the sentences

- B1: exp:11 (workloads, and that they changed); exp:16 (definition of AllOpt).
- B2: exp:78 "Disk- and network-heavy ..."; exp:80 "In contrast, in-memory databases ..."; "Chromium is the
  extreme case"; caption and legend of Fig. 7.
- B3: both sentences of exp:91. B5: exp:97 second sentence. B6: all three sentences of exp:100. B7: exp:103-104.
- B9: exp:107-109 entirely.
- B10: exp:115-116; the appendix paragraph; the "peak" clause in abstract, introduction, conclusion.
- B11: exp:127, 132, 134; abstract:24-25. B12: exp:143-148. B13: exp:156-158; abstract:26-27; intro:150;
  conclusion:16-17. B14: exp:162-166.
- B15: "fully automatic" (abstract:27, intro:149); intro:74-75; "geometric-mean Chromium speedup".
- B16: exp:86-88 and the box at exp:190-192.

Decisions that gate wording (11 `\staleclaim`, 3 `\stale`, 3 floats): the base of the speedups (X2); how
reduction is presented (B11); the description of the run-time forms and with it "fully automatic", "never miss
races" and AllOpt's definition (plan item 11); SQLite's metric (B7); the design of the Chromium figures (B8,
B9); the design of the contention figure (B10).

### (4) Page budget

The plan states that with the marks off the body ends on page 13, where the references start, and three
requested additions (bounded shadow state, preservation protocol, DE usability) are not written yet. So every
addition needs a cut. The evaluation occupies pages 8-13 with seven floats. Weak candidates after the new data:

| candidate | why weak now | possible saving |
|---|---|---|
| Fig. 9 (full width; MySQL select, MySQL write, two Chromium benchmarks) | one of four panels has data | the whole float, if MySQL becomes a fifth panel of Fig. 8 |
| Fig. 10 (Chromium box plot per suite) | no per-suite data; Chromium is five cumulative numbers | replace by one sentence or a five-row table |
| Fig. 11 (full width; SIR, DIR, DMIR, memory) | DMIR not re-measured; SIR is 0.6-12.4 %; memory is negative in four of five; DIR is another metric and "current" | one compact table (per application: checks removed by the analyses, drop in runtime entries, memory, compile time, code size) replacing Fig. 11 and Fig. 12 |
| Fig. 12 (compile time) | values of 25-189 % read as a weakness in a bar chart | a column of the same table |
| exp:136-138 (DMIR paragraph) | no data | 5 lines |
| exp:143-148 (memory paragraph) | one sentence of content left | 5 lines |
| exp:100 last sentence; exp:109 | contradicted or empty | 3 lines |
| Fig. 6 (full width, metric definitions) | DMIR and possibly SIR rows go | a few lines; could become a single-column table |
| thread-count paragraph and appendix figure | SQLite has no series; the result is "gain falls with threads" on one application | main text is 3 lines; the appendix is outside the limit |

Blocks that will grow: B7 (SQLite's metric and its caveats), B13 (spec generation and the whole-program step),
B14 (the preservation protocol), B1 (workloads and run counts), and the description of the run-time forms
(plan item 11).

## 4. Not checked

- Nothing was compiled; page counts and float positions are from the plan and from an `.aux` of 13 Oct.
- OS release and LLVM base version of the two hosts and of the compiler roots (the methodology branch states
  Ubuntu 24.04 and Clang 19).
- Whether per-story Chromium records exist in a form that yields per-suite medians and a distribution.
- Which DE form the timed DE and DE+Peeling trees of FFmpeg and SQLite were built in; and whether SQLite's
  record trees contain DE and peeling at all (two sources disagree, B7).
- Whether 1.286 and 1.248 are geometric means over stories.
- The raw leg records behind PR; PR and OR were taken as given. `experiments.md` and `camera-ready-mapping.md`
  were read in part only.
- Sections 3-5 of the paper, the proofs, the DE and bounded-shadow text (out of scope), the bibliography, and
  the upstreaming statements.
- The rendered appearance of any figure after a swap (axis limits, label collisions with nine bars).
