# Soundness fixes since the March 2026 compiler, and what they cost

29 Sep 2026. Anything inferred is marked *(inference)*.

## Short version

- **What was timed.** ThreadSanitizer (TSan) puts a check before every memory access. The optimizing compiler
  removes the checks its static analyses prove unnecessary. **Speedup** is the optimized build's speed over stock
  (unmodified) TSan's. Only four compilers were timed, never a single fix: **A** the March compiler, **B** after the
  first review, **C** after reviews 2-3, **D** the shipped compiler.
- **What a shape is.** A shape is a small program on which stock TSan reports a race and the optimized compiler does
  not. Shapes are numbered 1-62 in the order found. The fix that cost the most (item 1) carries no shape number.

**1. DE counted a check on one field as covering another (first review, step A→B).** DE (dominance elimination)
drops a check when an earlier check on the same location always runs before it, with no synchronisation in between.
- *Wrong:* "same location" meant the same object, whatever the field, index or size. DE counted `p->a` as covering
  `p->b`, `a[i]` as covering `a[j]`, and a 1-byte check as covering an 8-byte write.
- *Fix:* the earlier check must be provably at the same address and cover at least the same bytes.
- *Static effect of this fix alone:* DE's share of removed checks fell from 32.5 % to 2.9 %. That reach was unsound
  and cannot come back.

**What step A→B cost.** It holds fix 1 plus the first review's smaller fixes, and was timed only as a whole:

| app | speedup A → B | share of the app's total loss |
|---|---|---|
| SQLite | ×1.48-1.51 → ×1.09-1.10 | about three quarters (the rest: B→C 18-22 %, C→D 3-4 %) |
| Redis | ×1.41 → ×1.10 | nearly all (later steps: ×1.10 → ×1.09) |
| memcached | ×1.11 → ×1.02 | most (later steps: ×1.02 → ×1.00) |
| FFmpeg | ×1.356 → ×1.018 | all (D is back at ×1.018) |

Across the four apps, A→B is 75-100 % of the total loss. That fix 1 carries most of it is *(inference)* from
static counts: DE lost 35,004 of its 36,265 removed checks in this step.

**2. Escape analysis made fail-closed (reviews 2-3, step B→C).** EA (escape analysis) drops checks on memory that no
other thread can reach.
- *Wrong, for example:* the arguments of a function reachable through a pointer or from another unit were judged
  from the visible calls only. In `void put(int *p) { *p = 1; }` the write lost its check even when another unit
  passed a shared pointer. A pointer EA could not resolve was simply ignored.
  The rule is narrow: a `static` function whose address is never taken still gets its arguments' status from all
  its call sites (EA's interprocedural top-down pass), so `static void put(int *p)` called only with local pointers
  keeps its check removed. Only when some callers are invisible (the function is visible to other units in a
  per-unit build, or its address is taken as a callback or table entry) does its argument count as escaped. In
  whole-program mode all callers are visible; that relaxation was measured at ≈ 0 on SQLite.
- *Fix:* such arguments escape, and an unresolved pointer makes everything escape. An access before an object's
  publication is dropped only if every later escape is release-like. The same step made STC (single-threaded
  context) and SWMR (single writer, multiple readers) judge each unit on its own.
- *Measured:* SQLite ×1.09-1.10 → ×1.01, which is 18-22 % of its loss. The other apps move by at most 0.02 (Redis
  ×1.10 → ×1.09, memcached ×1.02 → ≈×1.01-1.02, FFmpeg ×1.018 → ×0.996). That EA carries SQLite's part is
  *(inference)*: EA lost 10,534 removed checks in this step, more than any other analysis.

**3. The rest before shipping (step C→D).** 3-4 % of SQLite's loss; memcached ×1.01-1.02 → ×1.00. By static counts
this step is almost all STC's rule that an unknown call may start a thread (`main(){ start_worker(); g = 1; }`).
Which fix costs SQLite's part is not known.

**After shipping** (the fixes for shapes 24-62 and exact DE; 60-62 were fixed on 30 Sep): each fix costs at most
1 percentage point of executed checks and no time that can be resolved, except possibly 2-3 % on Redis from exact
DE (unresolved).

## 1. Summary

This file brings together the fix inventory, the two lost-race ledgers, the results file and the checkpoint
timings listed under Sources (section 8). Every fact comes from those sources.

The March 2026 compiler (checkpoint A, 6 Mar) built the March binaries behind the submitted paper's measurements;
memcached without DE was built by a compiler of 3 Mar. It removed TSan checks with five static analyses: DE, EA,
STC, SWMR and LO (lock ownership). Section 2 explains what each one does. Since then, reviews, independent audits
and end-to-end runs have shown that it could **lose races**: stock (unmodified) TSan reports a race and the optimized
build does not.

Every lost race reproduced end to end is recorded as a numbered **shape**. There are 62 so far:
- **Shapes 1-23** were found before the paper compiler shipped (D), and all are fixed in it.
- **Shapes 24-62** were found after. 34 are fixed, 2 are excluded by stated premises (32 and 41), and 3 more (60-62, all EA) were fixed on 30 Sep.
- **LG-2**, a runtime case with no number, is fixed by exact DE.

The largest defect, DE's "same location" test, fixed in the first review, has no shape number.

**Cost.**
- **A→B took almost all of the speedup.** This first step is the first review:
  - SQLite ×1.48-1.51 → ×1.09-1.10;
  - Redis ×1.41 → ×1.10;
  - memcached ×1.11 → ×1.02;
  - FFmpeg ×1.356 → ×1.018.

  That step is 75-100 % of each app's total loss. By static counts, most of it is DE's same-location rule. That
  split is an *(inference)*: no single fix was timed on its own.
- **B→C and C→D cost noticeably only on SQLite.** B→C covers reviews 2-3, per-unit STC/SWMR and EA's six-shape
  commit; it is 18-22 % of SQLite's loss. C→D is 3-4 % of SQLite's loss.
- **At D the full configuration is at ×1.00-1.02 over stock** where D was timed (memcached, FFmpeg). SQLite is at
  ×1.01 and Redis at ×1.09 at C.
- **After shipping, each fix costs at most 1 percentage point (pp) of executed checks and no time that can be
  resolved.** The one possible exception is exact DE on Redis, perhaps 2-3 %, which is unresolved.

## 2. How to read

- **Terms.**
  - **Stock TSan** is unmodified ThreadSanitizer, with one upstream fix that the project's fork point
    (c609043dd009) predates. Upstream omitted an access when the capture query was false for an address whose
    underlying object is a stack alloca; the query was asked about the address value, so a field or element of a
    local aggregate counted as never captured even after the aggregate's address had been passed to a call or
    stored to a global. Upstream 59b26abbbe89 (24 Apr 2025, #132756; with bf6986f9f09f, #132752, before it) asks
    about the alloca instead, and the project's tree has had the same rule since 14 Sep. The fork point without
    the fix checks 3.5 % fewer accesses on memcached (6,382 against 6,604 calls) and is 3 % faster there, by
    missing those races, so it is not used as a baseline.
  - The compiler puts a **check** (a runtime call such as `__tsan_write4`) before every memory access.
  - The runtime records each access in **shadow memory**. Shadow is kept per 8-byte **granule** and holds a bounded
    number of records per granule. The runtime reports a race when two threads' accesses conflict with nothing
    ordering them.
  - Each analysis below proves that some checks are unnecessary and removes them.
- **Lost race.** A race that stock TSan reports and the optimized build does not. The project's rule is zero lost
  races, in every configuration.
- **Shape.** A minimal program that exhibits one lost race.
  - It gets a number only after an end-to-end reproduction. "Stock 20/20, EA 0/20" counts the runs, out of 20, that
    reported the race.
  - Every fix has a negative test that fails on its parent commit, and a positive twin in which the analysis still
    removes the check.
  - **Gated** means the fix also passed the IR test suites (IR is LLVM's intermediate representation), TSan's
    runtime tests (`check-tsan`, run in 12-16 configurations), the reproducers, and race-preservation runs, which
    check that the races stock reports on SQLite and memcached are still reported.
- **The analyses:**
  - **DE**, dominance elimination. DE drops a check when another check on the same location (the **cover**) runs
    before it on every path (**dominance**) or after it on every path (**post-dominance**), with no synchronisation in
    between. Stock TSan would report the race at the cover. **Loop peeling** copies a loop's first iteration, so that
    checks in the body have a dominating cover.
  - **EA**, escape analysis. EA drops checks on memory that never becomes reachable from another thread, for example
    a local variable whose address is never published.
  - **STC**, single-threaded context. STC drops checks in code that can run only before the program starts its
    second thread.
  - **SWMR**, single writer, multiple readers. If every write to a global happens in single-threaded context, reads
    of it need no check.
  - **LO**, lock ownership. If every multi-threaded access to a global holds one common lock, those accesses need no
    check.
  - **DynSTC**, run-time single-thread mode. The runtime counts live threads, and checks are skipped while the count
    is 1. It comes in two forms: a compiler-emitted guard, and a runtime form (`dynstc_rt=1`).
- **Checkpoints on the artifact line** (branch `artifact/atc26`):
  - **A** (6 Mar): the March compiler.
  - **B**: after the first review.
  - **C**: after reviews 2 and 3 and the per-unit STC/SWMR change. C is itself EA's six-shape fix, the last commit
    of that step.
  - **D**: the shipped compiler, also called the paper compiler. It sits 28 commits after the last March commit
    (10 Mar).
- **After D.** The fixes were integrated in stages, ending in P1-v3:
  - fixes collected on local branches;
  - P1 (25 Sep);
  - P1-v2 (26 Sep);
  - **P1-v3**, which landed on 26 Sep as `integrate/p1-0926`. It is the shipped configuration (EA,
    LO, STC, SWMR and DE with loop peeling) with DE made exact and with the soundness fixes found after shipping,
    up to shape 59. It is the reference for all new measurements.
- **Speedup.** The speed of the full configuration, "AllOpt with loop peeling" (the five static analyses together,
  with peeling), divided by the speed of stock TSan built at the same checkpoint. A value above 1.00 means faster
  than stock.
- **Sites.** The static count of checks in compiled code.
  - The **corpus** is 28 IR modules: sqlite3.c, SQLite's shell.c and memcached's 26 modules.
  - A **removed share** is the number of sites removed, relative to stock built by the same compiler.
  - **Executed checks, pp** is the change in the share of checks executed at run time, in percentage points,
    estimated from profile runs.
- **Fix IDs.** DE-1, EA-7, LO-4/4b and so on come from the fix inventory and the audit ledger.
  `optimization-results.md` reuses DE-1…DE-10 and STC-1…4 for unrelated optimization ideas.
- **Steps.** A→B, B→C and C→D are the checkpoint steps above. "After D" means after shipping.

## 3. Speedup at each checkpoint

**Table 1. AllOpt with loop peeling over stock TSan, per checkpoint**

| app | A (March) | B (first review) | C (reviews 2-3) | D (shipped) |
|---|---|---|---|---|
| SQLite | 1.48-1.51 | 1.09-1.10 | 1.01 | not given; C→D is 3-4 % of the total loss |
| Redis | 1.41 | 1.10 | 1.09 | not given |
| memcached | 1.11 | 1.02 | ≈1.01-1.02 | 1.00 |
| FFmpeg | 1.356 | 1.018 | 0.996 | 1.018 |
| MySQL | not measured per checkpoint | | | |

Key to table 1:
- **SQLite** is threadtest3's stable subtests: walthread1, walthread2, checkpoint_starvation_1 and stress2. The noisy
  subtest stress1 (run-to-run variation about 20 %) went from ×3.18 at A to ×0.55 at D.
- **FFmpeg** was read on the lab host; the workload is transcodes of one film. At A, copy (the single-threaded
  stream copy) was ×1.46 and mjpeg (an encoder) ×2.41.
  - Stock TSan's own speed differs slightly between checkpoints. Relative to D it is 0.981 at A, 1.023 at B and 1.024
    at C, so the runtime and stock instrumentation are not identical across checkpoints.
  - Stock's corpus sites also grow by 2,559 at C (table 4b, "Upstream fix").
- **Where the loss goes (measured).**
  - A→B is 75-100 % of each app's total loss.
  - B→C shows only on SQLite, at 18-22 % of its loss.
  - C→D is 3-4 % of SQLite's loss.
- **The submitted paper printed larger gains.** Over stock TSan it gave SQLite +71 %, memcached +7 %, Redis +45 %,
  FFmpeg +57 % and MySQL +16 % (Select) / +11 % (Write-only) (`optimization-results.md`, table 3).
  - Column A is the March compiler re-timed on today's bench.
  - Its gap from the paper's figures reflects the measurement setup, not the fixes *(inference: same compiler)*.
- **Raw cells** are in `fixcost-2026-09-24/`, and for A in the directory named after the March compiler, both in
  the lab host's `/extra` data area.

**Table 1b. Static reach on the corpus: removed sites per analysis, and the change at each step**

| analysis | A | A→B | B→C | C→D | D |
|---|---|---|---|---|---|
| DE | 36,265 (53.8 %) | −35,004 | +69 | −14 | 1,316 (1.9 %) |
| EA | 13,400 (19.9 %) | −1,671 | −10,534 | −62 | 1,133 (1.6 %) |
| STC | 3,234 (4.8 %) | −174 | −548 | −2,506 | 6 |
| SWMR | 2,397 (3.6 %) | −1,676 | −631 | −90 | 0 |
| LO | 733 (1.1 %) | −639 | −48 | −37 | 9 |
| AllOpt, no peeling | 44,862 (66.5 %) | −28,713 | −11,153 | −2,575 | 2,421 (3.5 %) |
| stock sites | 67,440 | | +2,559 | | 69,999 |

Key to table 1b:
- **How the step columns were made.** They are sums of the inventory's per-commit rows, with each commit placed in
  its step. The C→D column includes five commits the inventory lists together as "others". Placing them in C→D is an
  *(inference)* from their order and dates.
- **Percentages** are the removed share of stock sites.
- **DE's A→B** includes the first review's −20,680. The same-location rule alone takes DE from 32.5 % to 2.9 %
  (measured on a compiler of 10 Mar).
- **With loop peeling**, the corpus share is 61.5 % at A and −7.2 % at D. Peeling duplicates code, so once little is
  removed the peeled build has more checks than stock.
- **Static loss and time do not track each other.** EA's B→C loss of 10,534 sites cost little time outside SQLite.
  The inventory's explanation is that checks the analyses can soundly remove mostly take the runtime's fast path,
  about 15 cycles.
- **The first-review step, A→B, in shares of checks removed:** SQLite 59 % → 8 %, FFmpeg 76 % → 22 %.

## 4. The fixes, by analysis

Key to every table in this section:
- **Step** says where the fix entered:
  - "C→D\*" means the commit is one of the "others" whose placement is inferred;
  - "before D" means the inventory names no artifact commit for it.
- **Shapes** gives the numbers of the lost-race shapes the fix closes. "—" means the defect was found in review and
  carries no number.
- **Cost** is the static reach lost, or "0" when counts are identical. Reach is lost either as "+N sites" (N more
  checks kept) or as "−N removals" (N fewer checks removed). Time was measured only per checkpoint (table 1).
  "Not separated" means the fix's cost is inside its step's total.

### 4a. DE: dominance and post-dominance elimination (DE-1…DE-6; shapes 11, 12, 16; after D 25-29, 37, 38, 40, 44, LG-2)

| fix | what was wrong | what the fix does | shapes | step | cost |
|---|---|---|---|---|---|
| Same location | "Same location" meant the same base object or must-alias, with no size or offset check. A check on `p->a` covered `p->b`, `a[i]` covered `a[j]`, and a 1-byte check covered an 8-byte access. | `locationCovers`: must-alias with a covering size, or the same pointer value plus the same constant offset. | — | A→B | Alone, 32.5 % → 2.9 % (30 of DE's 32.5 points). That reach was unsound and cannot be recovered. The sound remainder, byte-range containment, was closed on 24 Sep at 2 accesses. |
| All effects on all paths | The scan stopped at the first dangerous instruction. The lock kind was reset at each call, so acquire-then-release counted as a pure acquire. | `scanPaths` unions every effect on every path. | — | A→B | not separated |
| Loop back edge | For a removed access inside a loop, the path around the back edge was not scanned, so a release later in the body was missed. | Re-scan the whole loop body. | — | A→B | not separated |
| Bodiless and `invoke` callees (DE-1) | A function without a body counted as non-synchronising. Callees of `invoke` (a call with an exception edge) were not scanned. | A bodiless function synchronises, and `invoke` is scanned like `call`. | — | A→B, C→D\* | not separated |
| Post-dominance termination (DE-3) | Post-dominance assumed that library calls return, so the later cover it relied on might never run. A helper's loop test was inverted (`isContainsLoops`). | Every call on the path must be `willreturn` or loop-free by summary. A loop is allowed only with a computed trip count. | — | B→C, C→D\* | DE corpus count unchanged in review 2 |
| Intrinsics | Every compiler intrinsic was treated as safe to cross. | Only `nosync` intrinsics that touch no memory, or only their arguments' memory. | — | B→C | as above |
| Library table fails closed (DE-5, DE-6) | A library function not on a short blocklist counted as non-synchronising. That covered `write(2)`/`read(2)` (TSan treats them as release/acquire on the file), every file, directory and time call, and `qsort` whose comparator unlocks. | The default becomes "synchronises", with an explicit list of 273 sync-free names. `read`/`write`/`pread`/`pwrite`/`open` are barriers. | 11, 12, 16 | C→D\* | shell.c +10 sites (DE 6,302 → 6,312); memcached, sqlite3.c 0 |
| Summary plumbing, cover chains (DE-2, DE-4) | DE-2: the sync-free summary read its own pointer before it was built. DE-4: in a chain of covers (a covers b, b covers c), each link was not re-checked against the surviving root. | An explicit module analysis; chains are re-scanned against the root. | — | before D | DE-2: reach identical at -O2 |
| Post-dominance across a back edge | The next iteration's header store, through a pointer that has since advanced, "covered" this iteration's body store after an unlock. | Reject a cover whose path crosses the back edge of a loop containing the access, unless both pointers are loop-invariant. Reject on irreducible control flow. | 25 | after D | 0 |
| One granule | The runtime's 16-byte and unaligned entry stops after a race in the first granule, so a race at `buf+8` was never reported. | A covered access is at most 8 bytes, aligned, and inside one granule. | 26 | after D | corpus +16, Redis +31, FFmpeg +1,774 sites |
| Sizes 1/2/4/8 only | A 3-, 5-, 6- or 7-byte access was accepted as a cover, although TSan has no entry for that size and makes no call. | Only 1-, 2-, 4- and 8-byte covers. | 27 | after D | 0 |
| No vptr cover | A C++ virtual-table pointer store was a cover, but `__tsan_vptr_update` checks only when the value changes. | vtable stores never cover. | 28 | after D | 0 |
| Irreducible cycles | Post-dominance's termination test saw only natural loops, so it missed a never-ending cycle with two entries. | Cycles are detected with LLVM's CycleInfo. | 29 | after D | 0 |
| Callee that never returns | A sync-free callee with a two-entry `goto` cycle, or in mutual recursion, counted as returning. In `r = x; spin(flag, r); x = 2;` the cover `x = 2` never runs. | A function counts as looping if it has an irreducible cycle or is recursive. | 38 | after D | 0 (DE byte-identical) |
| Names only for declarations | Calls were classified by name before their body was examined. A program's own `lock()` built on a condition variable (acquire plus release) was taken for a pure acquire (37). A program's own `strlen` that locks was read off the library table (40). Weak definitions were trusted. | A name counts only for a declaration, and the tables keep only names a program cannot redefine (POSIX, C11, reserved, mangled `std::`). A replaceable definition synchronises and may not return. | 37, 40 | after D | 0 |
| Weak members of recursive cycles | This is shape 40 through recursion: members of a recursive cycle were skipped before classification, so a weak one, replaced at link time by a version that unlocks, went unseen. | Weak members are not skipped, and attributes inferred over a replaceable body are not trusted. | 44 | after D | 0 (DE byte-identical) |
| Exact DE, "verified removal" (`-tsan-de-verified`; in P1-v3) | A removal relied on the cover's shadow record surviving until the removed access, but the runtime drops records. LG-2: a global shadow reset after about 4.2 M releases. F2: ordinary eviction by the thread's own writes to the same granule. Stock re-records at the removed access and reports; DE did not (10/10 vs 0/10). | The covered access keeps TSan's own hit test inline, identical to stock's check up to its first branch, and calls the runtime on a miss. Post-dominance, merging and exit ranges are off. | LG-2 | after D | section 5 |
| Signal delivery | TSan runs a pending signal handler inside intercepted calls. A handler that posts a semaphore is a release inside a call DE treats as sync-free. | No fix: premise A3. | 41 | — | — |

Key to table 4a:
- **Must-alias**: the compiler proves that two pointers are equal.
- **LLVM attributes.** **`willreturn`** means the function always returns, and **`nosync`** means it does not
  synchronise.
- **Control flow.** A **back edge** is the jump from a loop's end to its start. **Irreducible** control flow is a
  cycle with more than one entry.
- **DE's extensions.** **Merging** replaces adjacent checks with one wider check. An **exit range** is one range check
  for a whole loop. The **hit test** is TSan's fast-path test of whether the access is already recorded in shadow.
- **Rejected alternative.** A "generation guard", a run-time re-check when shadow was reset, was designed as the fix
  for LG-2. It was rejected because the eviction signal it needs would cost more than DE saves on SQLite.

### 4b. EA: escape analysis (EA-1…EA-13; shapes 1-6, 18, 19, 21-23; after D 24, 33-35, 39, 45-52, 55-59)

| fix | what was wrong | what the fix does | shapes | step | cost |
|---|---|---|---|---|---|
| Gaps against the escape definition (EA-1, EA-2, EA-3, EA-4, EA-9) | (1) `foo(&s); s.f1 = 1;`: the escape of `s` as a whole did not cover its field. (2) `&x` stored into a container that had already escaped. (3) `&x` published by compare-and-swap, with its reason bits truncated (with EA-4). (6) In an access through a loaded pointer whose object is published later, the rule walked the wrong object. | Lookups cover prefixes; the container's state is consulted; reason bits are widened; the later-escape walk follows the right object. | 1, 2, 3, 6 | B→C *(inference from C's commit title, "six shapes")* | inside C's −8,559 |
| Unresolved operands fail closed (EA-5) | `&x` stored through a pointer the object walk could not resolve: the operand was simply dropped. | If an unresolved operand is stored, passed or returned, every object escapes. | 4 | A→B, B→C | not separated |
| Slot with no known target (EA-6) | For a pointer loaded from a struct filled by `memcpy`, a slot with no recorded target answered "local". | It answers "escaped". | 5 | B→C | inside C's −8,559 |
| Address-taken and exported functions (EA-7) | Arguments of a function called through a pointer, or from another unit, were judged from the visible call sites only. | All their pointer arguments escape. | — | B→C | Alone +926 corpus sites. With the three fail-closed switches (unresolved operand, unseen target, sound flow-sensitive rule) +5,509. The commit as a whole is −8,559 EA removals, mostly in `EscapeAnalysis.cpp`. |
| Allocator and retention tables (EA-8) | An allocator was recognised by its name alone, and `setbuf`/`setvbuf` were not known to keep the buffer. | The prototype is checked, and setbuf/setvbuf retain the buffer. | — | B→C | inside C's −8,559 |
| Upstream fix (in C's EA commit) | Upstream TSan's "local not captured" test asked about a field's address rather than the variable's, so fields of a published struct went unchecked even in stock. Upstream fixed this in April 2025. | Ask about the variable. | — | B→C | stock +2,559 corpus sites |
| Flow-sensitive rule | Elision was decided per program point, so an access before the object's publication was elided whatever came later. | Such an access is elided only if every later escape is release-like (a thread creation, for example), and never for arguments. | — | B→C | EA −1,975 |
| Unknown library functions | An unknown library function was assumed not to let its argument escape. | It is assumed to let it escape. | — | B→C | 0 (the table names every function) |
| Out-parameter (EA-10) | In `f(&local, &slot)`, where `f` stores its first argument through its second, the escape carried only a bit that callers mask as "merely an argument". | A flag that survives the mask. | 18 | C→D\* | not measured separately |
| Arguments of a call whose result escapes (EA-11) | In `r = f(&local)`, where `f` publishes its argument and returns an escaped pointer, the arguments were never classified. | Only the callee operand answers "escaped"; the arguments are classified. | 19 | C→D\* | Redis +485, MySQL +9,673, FFmpeg +626 sites |
| Result points into an argument (EA-12) | In `r = strchr(b, c); g = r;` the result was treated as a new object, so publishing it did not publish `b`. The same held for strstr, memchr, strcpy, fgets and realloc. | The result points into the argument. | 21 | C→D\* | memcached, sqlite3.c 0; shell.c 10 fewer checks (a gain) |
| End pointer (EA-13) | `strtol(buf, &end, 10); g = end;` published `buf` unseen. | The call is modelled as the store `*end = buf`. | 22 | C→D\* | 0 |
| realpath, ctermid | In `g = realpath(p, b); b[0] = 'x';`, both functions were on the "does not escape" list. | The buffer escapes at the call. | 23 | C→D\* | 0 (neither name occurs in the corpus) |
| Copy from a non-local source | In a callee doing `memcpy(&g, c, n)`, where `c` is an argument or a heap pointer, whatever `c` points to was published unseen. | Every copy source is modelled. | 24 | after D | corpus +18 (sqlite3.c) |
| Caller context | The caller's escape state was consulted wrongly in three ways. (33) It was taken only at the block of the call: `helper(&x)`, then `g = &x` in a later block. (34) It was taken for `&s.b` rather than `s`. (55) It was taken from `nocapture`, which says nothing about what the argument's memory holds: `make_thing(&t)` stores a fresh object into `*out`, and `t` is then published. | Ask about the object anywhere in the caller; ask about the variable; use capture tracking only for an argument the call only reads. | 33, 34, 55 | after D | not measured separately |
| Stores through unseen pointers | (35) A callee stores `&local` through a pointer loaded from a pointer-array argument. (39) `set(&p)` repoints `p`, but the analysis kept the old target. | A stored object escapes when the slot's targets are unknown. At a call that may write through a pointer argument, the recorded targets escape. | 35, 39 | after D | not measured separately |
| Stop modelling memory contents | EA's model of what memory holds was unsound. (45) "Loaded" was lost through select, phi or integer casts. (46) `memcpy(d, s)` was modelled as `d` pointing to `s`; this was introduced by shape 24's fix and is not in D. (48) `nocapture` was taken to mean "contents do not escape". (49) One field's target vouched for another field. (50) A stored loaded value published nothing. (51) Two loads counted as one. | An access through a loaded pointer escapes, and a store through one publishes. A stored loaded value publishes its source slot when the destination is shared. Every copy source escapes. Library names apply to declarations only. | 45, 46, 48-51 | after D | All EA fixes of shapes 33-51 together, at the A2 gate: corpus EA +356 sites (shape 48 +86; the loaded-contents commit +62); executed +0.00 pp on SQLite and memcached |
| Weak callee result | A weak `pool_get` returning `malloc(16)` made its result local, but the strong `pool_get` actually linked returns a shared buffer. | A callee without an exact definition has an unknown result. | 47 | after D | not measured separately |
| Integer-copy closure (`-tsan-ea-integer-copies-escape`, on by default since 25 Sep) | A pointer's bytes copied as a 64-bit integer were not followed. | Such copies make their targets escape. This replaces premise A7, which was not adopted. | — | after D | section 5 |
| First receiver only (E1, in P1) | The flow-sensitive rule elided an access before publication when every later escape was release-like, but the receiver then passed the object on, by a relaxed store, to a third thread. | Elide only when no escape is reachable after the access. The old rule stays behind `-tsan-ea-trust-first-receiver`, off by default. | 52 | after D | SQLite 0 static |
| Linkage of acquire-only globals (E2) | An external global read here only by acquire loads could be read relaxed in another unit. | Local linkage is required. | — (did not reproduce) | after D | not measured separately |
| Self-store published later | `init_self(&x)` stores `&x` into `x.self`, and a later block publishes `x.self`. | A self-store through memory other code sees publishes the object. | 56 | after D | not measured separately |
| setjmp's second return | Block states follow control-flow edges, and setjmp's second return has none. | EA gives up (everything escapes) in a function that calls a returns-twice function. | 57 | after D | not measured separately |
| Pointer vectors, unresolved callee operands | A vector of pointers stored by a masked store published nothing (EA-P3: IR level only, needs AVX2 vector code). In a callee, an operand the walk could not resolve never reached the callee's argument summary (58). | Operands whose type holds a pointer are examined. An unresolved operand anywhere makes every argument escape. | 58 | after D | 0 sites, 0.000 % executed (SQLite, memcached) |
| `main` in whole-program mode | Under `-tsan-whole-program`, `main` had no callers, so `envp` stayed local, although it is the same array as `environ`. | `main` counts as called from outside the program. | 59 | after D | whole-program mode only |
| Pipe, `%p` text, generic atomics (EA-P5, EA-P6, EA-P7) | A slot holding `&l` is written to a pipe (60), formatted as `%p` text (61), or copied by `__atomic_load` (62), and another thread writes `l` through the pointer. | Fixed 30 Sep: a call operand escapes when the callee may send what it reads through it (or, if variadic, its value); copies are modelled as copies. | 60-62 | after shipping (30 Sep) | ≈ 0 (memcached 0.000 %, FFmpeg 0.004 % of executed checks) |
| Use after free | A thread writes through a stale pointer into memory that has been freed and reallocated. | No fix: the no-use-after-free premise. | 32 | — | — |

Key to table 4b:
- **Escape vocabulary.** A **slot** is memory that holds a pointer, and its **targets** are what that pointer may
  point to.
  - **Loaded** means obtained by reading memory.
  - **`nocapture`** is an LLVM attribute: the callee keeps no copy of the pointer.
  - **Returns-twice** functions are setjmp-like.
- **IR terms.** A **phi** merges values arriving from different predecessor blocks, and a **select** is a
  conditional value.
- **Flow-sensitive rule.** An access to an object that escapes later may be elided only when the later escapes are
  release-like, because a release orders the access before any thread that receives the object.
- **The three fail-closed switches** in the EA-7 row are `-tsan-ea-unknown-operand-is-top`,
  `-tsan-ea-unseen-pointee-escapes` and `-tsan-ea-sound-flow-sensitive`.
- **A2 gate.** The second gate of the post-shipping fix series (24-25 Sep).
- **Audit labels.** E1 and E2 are items of audit A1. EA-P1…EA-P7 are holes found by audits A8 and A17; each got a
  shape number once it was reproduced. Audits A1, A8, A17 and so on are numbered independent audits.
- **Whole-program mode** (`-tsan-whole-program`): the analyses run once over the whole linked program instead of
  one unit at a time.

### 4c. STC: single-threaded context (STC-1…STC-3; shapes 7, 15; after D 30, 31, 43, 54)

| fix | what was wrong | what the fix does | shapes | step | cost |
|---|---|---|---|---|---|
| Creator's callees (STC-1) | In `helper()` after `pthread_create()` in a function other than `main`, the thread creator's callees were not marked multi-threaded. | They are marked. | 7 | B→C | the commit: STC −840, SWMR −631 |
| Per-unit visibility (STC-2) | A function this unit never calls defaulted to single-threaded, which is valid only when the whole program is visible. | In per-unit mode, external and address-taken functions are multi-threaded. | — | B→C | as above |
| Unknown calls may create threads (STC-3, STC-3b) | `main(){ start_worker(); g = 1; }`, where the thread is started inside a call this unit has no body for (another unit, a library) or by an indirect call. A static constructor could also start a thread. | Any unknown external or indirect call may create a thread, except known library functions. A constructor that may create a thread makes `main` multi-threaded from its entry. | 15 | C→D | STC −2,506 corpus. memcached sites under STC 6,515 → 6,910, of stock's 6,911, so almost nothing is removed: the prefix now ends at libevent's first call. shell.c 4,336 → 6,447 (stock 6,452); 91 % of shell.c's former reach was in functions that really reach `pthread_create`. |
| setjmp/longjmp | A `longjmp` returns into `main`'s single-threaded prefix after `pthread_create` (30). `__builtin_setjmp` lowers to an intrinsic without the returns-twice attribute (43; shape 30's fix was incomplete). | Every setjmp-like call is a second return point. | 30, 43 | after D | 0 (corpus identical) |
| Reach outside the IR (also SWMR and LO) | A symbol with local linkage can still be reached in ways the unit's IR does not show: the linker's `__start_`/`__stop_` section bounds, an alias with external linkage, or assembler text. STC, SWMR and LO took "no other use in this IR" to mean "private". (LO case: a static mutex another unit releases inside an opaque call.) | One test, `isReachableOutsideIR`, used by all three. | 31 | after D | 0 everywhere (corpus, Redis, FFmpeg identical) |
| Thread created under ignore-sync (also SWMR, DynSTC guard) | `g = 1` in main's prefix, then `AnnotateIgnoreSyncBegin`, then `pthread_create` of a thread that reads `g`. TSan's thread creation releases nothing inside such a region, so stock reports a race. | No conclusion in a unit that opens such a region. | 54 | after D | not measured separately |

Key to table 4c:
- **Prefix.** The part of `main` before any thread can exist.
- **ignore-sync region.** Code between the annotations that tell TSan to ignore synchronisation.
- **First-review cost.** The first review also cost STC 174 removals; the inventory does not say
  why.

### 4d. SWMR: single writer, multiple readers (SWMR-1, SWMR-2; shapes 8, 20; after D 31, 42, 54)

| fix | what was wrong | what the fix does | shapes | step | cost |
|---|---|---|---|---|---|
| Other uses count | Uses other than load and store were skipped, so `strlen(g)`, `printf("%s", g)` and `memcpy` could hide a write or a publication. | Any other use counts as an escape. | — | A→B | SWMR −1,676 in that commit |
| Local linkage only (SWMR-1) | An `extern` global was read here and written in another unit (memcached's `current_time`). | Only globals with local linkage qualify, so memcached's `settings` and `current_time` drop out. | 8 | B→C | in the commit's −631 |
| Address in an initializer (SWMR-2) | In `@slot = global ptr @g`, a thread loads the pointer and writes `g`, yet `g` counted as read-only. | This counts as an escape. | 20 | C→D\* | 0 |
| Self-address store (also LO) | `head.next = &head` writes the global and publishes its address in one store, but only the store's pointer operand was examined. | A store whose value is the global's own address is an escape. | 42 | after D | not measured separately |
| Reach outside the IR; thread under ignore-sync | See table 4c. | | 31, 54 | after D | 0; not measured separately |

Key to table 4d: **local linkage** means that no other unit can name the global.

### 4e. LO: lock ownership (LO-1…LO-8; shapes 9, 10, 13, 14, 17, 20; after D 31, 42, 53)

| fix | what was wrong | what the fix does | shapes | step | cost |
|---|---|---|---|---|---|
| Lock identity | Locks were identified by name. | A lock is identified as a global plus a constant offset. | — | B→C | LO −417 |
| Exact lock names (LO-1) | Any name containing "lock" was an acquisition ("block" contains "lock"). | Exact tables of lock functions. | 10 | C→D | the commit (LO-1, LO-4/4b, LO-2): corpus LO −37 |
| Private, unescaped globals only (LO-4/4b) | Any global counted, including extern ones that other units access. | Only globals with local linkage whose address never escapes. | — | C→D | memcached, LO alone: 46 → 9 elisions (its mutexes are external globals) |
| Callee and opaque-call releases (LO-2) | A callee that releases the caller's lock was not modelled, and an opaque call was assumed to release nothing. | Release summaries over the call graph. An opaque call releases every non-private mutex. | 9 | C→D | in the commit's −37 |
| Held set from an intermediate visit (LO-3/5) | A store reached from a lock-free path counted as protected, because the held-lock record came from an intermediate fixpoint visit. | Blocks are seeded in reverse post-order, and the record is replaced on every visit. | 13 | before D | not measured separately |
| Timed and try locks (LO-6) | `pthread_mutex_timedlock`/`trylock` counted as an acquisition on both branches, although the timeout path holds nothing. | Held on neither branch. | 14 | C→D\* | 0 (no timed locks in the corpus) |
| OpenMP fork (LO-7) | The fork's microtask also runs on the calling thread and may release the caller's lock. | Such routines count as possible releasers. | 17 | C→D\* | 0 |
| Address in an initializer (LO-8) | As SWMR-2. | As SWMR-2. | 20 | C→D\* | 0 |
| Locks TSan does not intercept (in P1) | LO counted C11 `mtx_lock` and OpenMP `omp_set_lock` as locks, but TSan intercepts neither. Stock therefore orders nothing and reports the race, while LO silenced it. The exact tables of LO-1 had included them. | Only pthread mutex, spin and rwlock names. | 53 | after D | not measured separately |
| Reach outside the IR; self-address store | See tables 4c and 4d. | | 31, 42 | after D | 0; not measured separately |

Key to table 4e:
- **Terms.** An **opaque call** is one whose callee is unknown. A mutex is **private** if it has local linkage and is
  used only as a mutex operand.
- **Review 2.** The second review *added* 369 LO removals, and the first review removed 639. The
  inventory does not describe either change.
- **Rejected precision switches.** Three switches on `experiment/lo-precision` were never shipped. LO-B1 and LO-B2
  were found unsound (they lost races) and LO-C gained nothing.

### 4f. DynSTC, the pass and the runtime (PASS-2, PASS-3, RT-1…RT-3, NAMES-1; after D 36, 54)

| fix | what was wrong | what the fix does | shapes | step | cost |
|---|---|---|---|---|---|
| Join without acquire | A `pthread_join` inside an ignore-sync region acquires nothing, yet the runtime decremented the live-thread count, so main's later write was skipped. | Decrement only when the acquire ran. | 36 | after D | not measured separately |
| Guard under ignore-sync | See table 4c. The compiler guard is not emitted in such a unit. | | 54 | after D | not measured separately |
| Interceptor toggles (PASS-3, RT-1) | Skipping a string or memory interceptor for local operands was allowed for calls that can unwind, and without the flow-sensitive escape rule. | Allowed only for calls that do not throw, under the sound rule. | — | before D | not measured separately |
| Function entry/exit kept | Fully single-threaded functions dropped `__tsan_func_entry/exit`, which maintain the call stack shown in reports *(inference)*. | They keep them. | — | before D | the only item in this group the inventory calls noticeable in time; not quantified |
| Summaries only on request | Not described. The fix implies that whole-program summaries were read without the flag. | Read only under `-tsan-use-analysis-summaries`. | — | before D | not measured separately |
| Research code off (RT-2, RT-3) | The ReX filter and the eviction counters were compiled in. The counters made stock TSan on the development tree about 1.4× slower. | Both are behind build options, off by default. The access code is then byte-identical to upstream. | — | before D | affects stock, not the analyses |
| One creator list (NAMES-1); PASS-2 | Two lists of thread creators; a null-pointer hygiene fix. | | — | before D | — |

Key to table 4f:
- **Interceptor.** TSan's wrapper around a libc call.
- **ReX.** A research race-filtering component in the runtime.
- **DynSTC runtime form.** `dynstc_rt=1` cannot see ignore-sync regions. It relies on a stated premise (section 6,
  A10).

## 5. After shipping: fixes and their cost

**Table 3. Cost of the fixes made after D**

| fix | shapes | cost |
|---|---|---|
| Exact DE, verified removal | LG-2 (and F2) | P1-v3 over D: SQLite 1.003, memcached 1.01, MySQL 1.00-1.03, FFmpeg 0.997, Redis 0.97-0.98 (possible 2-3 % cost, unresolved) |
| Exact DE, effect on the DE package | — | It removed the gains of the inexact DE package (merging and loop ranges without verification) on FFmpeg: mjpeg 1.40 → 1.00, copy 1.09 → 1.03 |
| Integer-copy closure | — (A7 not adopted) | FFmpeg +0.86 pp of executed checks (AllOpt; +1.01 pp EA alone), +10.7k sites. SQLite +0.04 pp, memcached +0.00. No measurable time (MySQL 0.973, within noise). |
| Cover within one granule | 26 | static only: FFmpeg +1,774, Redis +31, corpus +16 sites |
| Shapes 24 and 33-51 (mostly EA), plus the fail-closed sync-free table | 24, 33-51 | ≤ 0.1 pp of executed checks, no measurable time. The series alone: FFmpeg +0.09 pp, SQLite and memcached +0.00. |
| DE, STC, LO, SWMR shapes 25, 27-31, 37, 38, 40, 44 | | 0 (static counts identical) |
| Shapes 52-59 (P1 to P1-v3) | 52-59 | 52: SQLite 0 static. 58: 0 sites, 0.000 % executed. Others not measured separately; all are inside "P1-v3 over D". |
| EA-P5/P6/P7 | 60-62 | Fixed 30 Sep after an independent audit: a pointer operand escapes when a call may send what it reads through it. Cost ≈ 0: memcached +19 sites (0.000 % of executed checks), FFmpeg +42 (0.004 %), SQLite and Redis 0. Whether D has them was not checked. |

Key to table 3:
- **Reading the ratios.** A ratio is the configuration with the fix over the same configuration without it. 1.000
  means no cost, and a value above 1 means faster.
- **What "P1-v3 over D" contains.** The results file gives the same figures as "P1-v3 over the shipped artifact"
  (SQLite +0.3 %, memcached +1.4 %, Redis −2.9 %, MySQL 0…+3 %, FFmpeg −0.3 %). They therefore contain every fix made
  after D, not exact DE alone. The other post-shipping fixes cost ≤ 1 pp of executed checks, so the figures mostly
  reflect exact DE *(inference)*.
- **Resolution.** Timing resolves about 2-4 % (MySQL about 5 %).
- **pp.** Percentage points of executed checks, estimated from profile runs.

## 6. Premises the soundness claims rest on

Each premise excludes a class of programs. In an excluded program, a race may be lost.

**Adopted:**
- **No use-after-free** (adopted 24 Sep). A race reached only through a dangling pointer to freed and reallocated
  memory, or to reused stack memory, may go unreported (shape 32).
- **A3, signals** (adopted 24 Sep; later clauses provisional, 25 Sep).
  - Signal handlers that synchronise with other threads are outside the model, and so is signal delivery itself
    (shape 41).
  - Stock TSan also sees such synchronisation only where it delivers the signal. For synchronous handlers, however
    (a SIGSEGV handler that posts a semaphore), A3 is a real exclusion.
  - Every `__tsan_atomic*` entry is also a delivery point, and a handler that does not return is outside the model.
- **A12, closed world** (adopted 25 Sep). No function is called from outside the program, except through a plugin
  API listed with `-tsan-external-symbols`.
  - Only whole-program mode relies on it.
  - FFmpeg's and Redis's whole-program numbers carry no soundness claim until their external-symbol lists are built.
- **Criterion G** (accepted 25 Sep). Merging under verified removal uses a hit rule that counts a covering record of
  the same thread and epoch (TSan's per-thread logical time), plus an absorbing store. With these, every granule
  stock reports on is still reported, no later, but not pair for pair.
- **R3 and P-OWN** (adopted 29 Sep). They apply only to LO-OBJ-G, a new optimization (lock ownership per owning
  object, with a run-time guard), not to the shipped analyses.
  - **R3**: every conflicting access to an annotated object holds the annotated lock, or happens before the object
    is published.
  - **P-OWN**: the annotations name the right lock.

**Stated premises, in use:**
- **P-EV**, the eviction victim is arbitrary. Every analysis relies on it, and the paper must state it. From 25 Sep
  to 1 Oct it was read strictly (shadow contents equal stock's at every point), which forced DE to verify each
  covered check at run time. Ruled 1 Oct: as in the paper's completeness remark and the ATC rebuttal, a race lost
  only because the covering record was evicted is accepted, so DE may remove covered checks outright.
- **ASYNC-TERM** (adopted 1 Oct, widened the same day). A race may go unreported if, between two accesses of one
  thread, the process or the thread is terminated asynchronously (another thread's exit or abort, SIGKILL,
  asynchronous cancellation), or a signal handler runs that does not return (exit, _exit, longjmp out of the
  interrupted code). It licenses terminating post-dominance in DE (loops need a computable trip count; calls must be
  willreturn and nounwind) and the loop-range form of DE, whose loops contain no call and no synchronisation.
- **P-RESET** (ruled 1 Oct), next to P-EV. A race lost because a global shadow reset (epoch exhaustion,
  `__tsan_flush_memory`, `memory_limit_mb`) dropped the record a removal rested on is accepted in the same class as
  eviction: a bounded-shadow state loss. It is LG-2 (`soundness-0924/lg2/lg2b_epoch.c`: stock 10/10, dominance DE
  0/10). Removal-mode DE, merge and loop ranges carry it. memcached resets about once a second (453 resets in a
  489 s stock run, slot census of 1 Oct), so there it is a frequent case, not a corner. The loop guard and verified
  DE re-check after a reset and do not need it.
- **A2** (provisional 24 Sep, adopted 1 Oct). An indirect call reaches only functions of the same IR function
  type. Calling through an incompatible function type is undefined behaviour in C, and it is the rule Clang's
  control-flow integrity enforces. The memcached event-loop confinement, the thread-root SWMR rule and the
  whole-program summaries rely on it. A program that calls a function through an incompatible pointer type, which
  works on common ABIs, is outside the model.
- **A2-LIB** (adopted 1 Oct). The program does not call a function pointer that only a library produced (returned
  from a library call, loaded from library-owned data, or passed by a library to a callback), other than pointers
  the program stored there itself.
  Address-taken library declarations are ordinary indirect-call targets and are handled. The thread-root SWMR rule
  needs it to bound what an indirect call can reach.
- **FWD-PROGRESS** (adopted 1 Oct). A loop that the language lets the compiler assume to terminate (C11 6.8.5p6,
  C++ forward progress: no input/output, volatile access, atomic or synchronisation operation in it) does terminate.
  The loop-range form of DE and terminating post-dominance take a trip count that LLVM derives under this rule.
- **P5** (pending since 27 Sep, adopted 1 Oct). A race report lost because the runtime recycled the trace part that
  held the other thread's history, during a deferred or range check, is a bounded-state loss of the same class as
  P-EV and P-RESET.
- **A10**, the parts no unit-local check can see: an ignore-sync region opened in another unit, or by the runtime
  around an ignored library. DynSTC's runtime form sees none of them.
- **A11**, no thread exists when `main` starts, except one started by this unit's constructors, which are checked.
- **Plain lock success.** A plain `pthread_mutex_lock` does not fail.
- **Library names.** A libc or libstdc++ name binds to that library.

**Provisional:**
- **A4** (24 Sep). No unit defines a reserved C or POSIX library name that another unit calls. Shape 40's fix covers
  only definitions the unit itself can see.
- **A5** (24 Sep). A default-visibility definition in position-independent (`-fPIC`) code is not replaced at load
  time.
- **A6** (24 Sep). An `available_externally` body equals the definition that gets linked. This matters only under
  link-time optimisation (LTO).
- **A8** (25 Sep). The runtime keeps its default `force_seq_cst_atomics=0`.
  - It is needed by `-tsan-de-relaxed-atomic-nosync` (DE crosses relaxed atomics) and by N1-ATOMIC (an inline hit
    test for relaxed atomics), and narrowly by DE through the `nosync` callees it trusts.
  - A start-up check has been written but not built.
- **A9** (candidate, 25 Sep). A fatal fault ends the process. Post-dominance DE uses it.

**Not adopted:**
- **A7**: pointer bytes are copied, never computed. The integer-copy closure is on instead.
- **A3-fiber**: a handler returns on the same fiber.
- **P-PAGER** (29 Sep): SQLite's Pager and Wal objects are accessed only under their B-tree's mutex. SQLite states no
  assertion for it, so it was declined.
- **P-HAND** (29 Sep): a Redis client's buffers are touched only by the main thread while the io threads are idle, or by
  the io thread that holds the client during a phase. It trusts the program's own thread protocol, a class not
  adopted; the run-time quiet-thread mode (2 Oct) replaces it without that trust.

**Relaxed (results labelled "relaxed", kept apart from race-preserving results):**
- **P-EV-DEFER** (decided 28 Sep, extended 29 Sep). It applies only to two relaxed DE flags, not to the shipped
  analyses: loop ranges (DE-3R, a sync-free loop's accesses checked as one range after the loop) and adjacent-field
  merging (DE-2R, neighbouring fields checked by one wider check). They may lose a race only when the other access's
  shadow record disappears between an access and its deferred or early, widened check, the kind of loss the bounded
  shadow already causes in stock TSan. The 29 Sep extension also admits an unhandleable termination (SIGKILL, power
  loss) inside a sync-free ranged loop. An independent audit found two further losses outside this class (a ranged
  loop whose exit is never reached; a group whose record is not stored after a report); their fixes are designed.

## 7. What is not known

- **Time per fix within a step.** Only checkpoints were timed, so the time cost of any single fix inside A→B, B→C
  or C→D is unknown.
  - That DE's same-location rule accounts for most of A→B is inferred from static counts.
- **Missing checkpoint values.** SQLite and Redis at D; MySQL at every checkpoint.
- **Placement of five commits.** The inventory lists five commits together as "others": the two commits of the
  library-table fix (the first also completes DE-1 and DE-3), and the commits of EA-10, EA-11 and EA-12 (the last
  also carries SWMR-2 and LO-8). Placing them in C→D is an inference.
  - The artifact commits of LO-3/5 (shape 13), DE-2 and DE-4 are not named.
  - The commits of LO-6 and LO-7 changed no counts, so their position is not recorded either.
- **The time cost of shapes 52-62's fixes.** 52-59 are timed only as a bundle, in "P1-v3 over D". 60-62's fix costs ≈ 0 by executed-check counts (not timed separately).
- **Whether D has shapes 52-62.** Shapes 52-59 were not run on D, and 60-62 were not checked.
- **Whether A has shapes 1-23.** Their presence in the March compiler is not recorded. Most of the code involved
  predates the audit *(inference)*.
- **Redis under exact DE.** The possible 2-3 % loss is inside the timing resolution.
- **The DE library table's cost.** The inventory records a static cost of 10 sites on shell.c and 0 elsewhere. It
  also mentions a lab note that claims a larger cost. The contradiction has not been settled by measurement.
- **"The fail-closed sync-free table" in Table 3.** The sources do not say which commit this denotes. It may be the
  declarations-only name tables of shapes 37 and 40 *(inference)*.
- **A known open issue with no reach.** DE trusts alias analysis across an `addrspacecast` that changes pointer size
  (x86 `__ptr32`; found by audit A19b, one of the numbered independent audits). No module in the five applications
  uses it. It is still to be stated as a premise or guarded.

## 8. Sources

Paths are relative to this repository unless stated otherwise.
- `archive-2026-09-25/soundness-fixes-and-recovery-plan.md`: the fix inventory by analysis (what was, what became,
  commit), the per-commit static counts from A to D, the decomposition of C's EA commit, and the recovery plan.
- `shapes-ledger.md`: shapes 24-62 and LG-2, their fixes and costs, whether D has them, the gates, the premises
  (no use-after-free, A2-A12, P-EV) and the rejected designs.
- `removed-from-artifact/2026-09-23-evening/TSanAnalysesAudit.md`: shapes 1-23, the function-level audit, the fix IDs
  and the static counts after STC-3, DE-6 and LO-7.
- `optimization-results.md` (state of 28-29 Sep): P1-v3, exact DE, P-EV, P5, R3 and P-OWN, "P1-v3 over the shipped
  artifact", and the paper's printed speedups.
- The checkpoint timings of 24 Sep: raw cells in `fixcost-2026-09-24/`, and for A in the directory named after the
  March compiler, both in the lab host's `/extra` data area.
