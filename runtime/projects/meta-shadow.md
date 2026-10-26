# Project 1: Ownership meta-shadow ("TSan Fast-Filter Experiments"): audit

State 26 Oct 2026. Read-only audit: nothing was built or run. Sources:
- Docs repo `/tmp/claude-1005/-home-alexey-tsan-instr-paper/2549303b-8746-4a90-895f-c3b8326ddf8d/scratchpad/tsan_filter`
  (branch feat/docs-reorganization, one commit b391b6b); cited below as `docs:<file>:<line>`.
- Code in `/extra/alexey/tsan-rt/repo`, all three branches on base 73e729f7b2f0 (upstream, 26 Aug 2026):
  - `meta-shadow fork: feat/shadow-resident-filter`: 2 commits (7c8ce9a14d78, c883c4524e3a), 9 files, +259/-2;
  - `meta-shadow fork: feat/parallel-note`: 2 commits (7bbf79754677, 3995d5da1c07), 12 files, +544/-5;
  - `meta-shadow fork: feat/workload-profiler`: the two parallel-note commits plus 8c9c383c792b, 19 files, +1545/-18.
  Cited as `<branch>:<path>:<line>` (line numbers in the branch's version of the file).
- Diffs kept in this folder: `sr.diff` (shadow-resident), `pn-rt.diff` (parallel-note runtime).

Hosts: most results are a "Linux/AArch64 VM" (`docs:docs/benchmarks/results.md:5-8`), host CPU not named; memcached and
MySQL "native Linux" x86-64 (`results.md:34-82`), host not named.

---

## A. Shadow-resident filter (`feat/shadow-resident-filter`)

### A1. Mechanism

- **What is stored:** nothing new. Before an eligible access, the compiler emits a replica of stock
  `ContainsSameAccess`. It builds the current 32-bit shadow word as `fast_state | (((1<<size)-1) << (addr&7)) | read<<30`,
  loads slot 0, and on a miss loads slots 1-3. It calls the normal `__tsan_{read,write}N` only if no slot matches
  (`shadow-resident-filter:llvm/lib/Transforms/Instrumentation/ThreadSanitizer.cpp:731-790`).
  - Writes need bitwise equality. Reads compare `old | kRodata` with `cur` and also accept `old == kRodata` (`:766-777`).
    This is the same rule as stock (base `tsan_rtl_access.cpp:173-191`, non-vector form).
- **Where the inputs come from:**
  - The fast state comes through a pointer returned by a new exported call, `__tsan_shadow_resident_fast_state()`
    (`compiler-rt/lib/tsan/rtl/tsan_interface.cpp:25-28`). The call is emitted once in the function's entry block
    (`ThreadSanitizer.cpp:709-718`).
  - The fast state itself is re-loaded at every access (`:753`).
  - The shadow address is computed from three exported globals: `__tsan_shadow_address_mask`, `__tsan_shadow_xor` and
    `__tsan_shadow_add` (`tsan_rtl.cpp:64-70`). They are set in `InitializeShadowMemory`
    (`tsan_platform_posix.cpp:61-64`) through new `MappingField` cases (`tsan_platform.h:914-916, 948-953`). The pass
    loads them in the entry block.
- **Eligibility** (`ThreadSanitizer.cpp:590-593, 706-708`):
  - Covered: aligned 1/2/4/8-byte plain accesses that are not volatile and not compound.
  - Only in functions with **no non-intrinsic call, no atomic and no mem-intrinsic** ("leaf" functions).
  - Targets: Linux or Darwin, little-endian, 64-bit (`:229-232`).
- **Validity, invalidation, sync, epochs, slots, eviction, reuse:** nothing is cached, so nothing needs invalidating.
  The test reads the live shadow and the live fast state, exactly as the runtime's own fast path would at that instant.
  Every release, slot change, reset, eviction, free marker or report clear is seen through the shadow and fast state
  themselves.
  - The ignore bit (bit 31 of the fast state) lands on the atomic bit of `cur`, as it does in stock (base
    `tsan_shadow.h` Shadow ctor: "kAccessAtomic overlaps with ignore_accesses_"). An ignored access can only match an
    atomic record and skip; stock skips ignored accesses anyway.

### A2. Soundness

**EXACT against stock** (same skip decision as stock's hit test at the same instant; no state change on a miss). I
found no scenario that loses a race.

Residual robustness points (not losses):
1. **Initialisation.** If an eligible function runs before `InitializeShadowMemory`, the three globals are 0 and the
   shadow pointer is 0, so the program crashes. No guard and no test (`ThreadSanitizer.cpp:715-718`). Not reachable in a
   normal executable (preinit), but untested on Darwin and with instrumented dylib initialisers.
2. **No layout check.** The pass hard-codes the FastState/Shadow bit layout (bit 30 read, low byte mask; `:757-758`).
   Unlike our N1 (`CheckInlineHitTestAbi`, run at every start, 8345a `tsan_rtl.cpp:831`), nothing verifies at run time
   that the runtime's layout matches the compiled code. A future format change (our R24 "wider epochs") would make
   stale objects skip silently. Upstream will ask for a version or layout check.
3. **Leaf-only restriction.** This is the reason for the pointer-at-entry design: a call could switch fibers, which
   changes `cur_thread()->current`. It is sound, but it is the main cost limiter (see A3).
4. **Tests have no negative control** (rule "negative reproducer control"):
   - `compiler-rt/test/tsan/shadow_resident_filter.cpp:1-44` has two cases. "private" checks only that the program
     runs. "race" is caught on a first-access miss in each thread, so it passes even if the inline path is wrong.
   - No test proves that a skip fired, or that a near-hit must miss: different mask or offset, read after a narrower
     write, the same access after a release, after `free`, after `__tsan_ignore_*`.
   - `docs:docs/testing.md:33-35` claims tests of "slot-0 hits, hits in slots 1-3, all-slot misses ... runtime
     fallback". The lit test does not contain them: the docs overstate the coverage.

### A3. Results as reported

| workload (host) | stock | shadow-resident | change | source |
|---|---|---|---|---|
| private loop, 300 M writes (VM) | 0.5488 s | 0.1718 s | -68.7 % | results.md:26 |
| NPB BT-W (VM, 4 thr) | 22.44 s CPU | 15.75 s | -29.8 % | results.md:95 |
| NPB BT-A (VM) | 471.11 s | 319.62 s | -32.2 %; NPB time -33.0 %, "95 % CI 28.05-37.69 %" | results.md:98, 112-115 |
| IS-W / IS-B / CG-W / CG-A (VM) | | | -2.6 / +0.2 / +1.0 / -0.5 % CPU | results.md:101-110 |
| Redis (VM) | 92,377 req/s | 91,843 | -0.65 % | results.md:27 |
| SQLite speedtest1 (VM) | 5.609 s | 5.599 s | -0.2 % | results.md:28 |
| memcached 1.6.29 (native) | 67,837 ops/s | 67,897 | +0.09 %, CPU -0.13 % | results.md:44-46 |
| MySQL 8.0.39 (native x86-64) | 13.65 TPS (MAD 0.22) | 13.30 (MAD 0.10) | **-2.56 %** | results.md:65-67 |
| FFmpeg h264 / h265 / mjpeg / copy (VM) | | | wall -0.2 / +0.9 / -1.2 / -5.1 % | results.md:130-141 |

Methodology:
- 1 warm-up and 7 interleaved runs per arm, medians. MAD is given for MySQL only; a CI is given for BT-A only.
- **No A/A** is reported, although the policy asks for one (`docs/benchmarks/README.md:115`).
- VM drift is admitted for H.265: stock wall-time SD 27 s (`results.md:146-147`).
- No CPU pinning is stated for the VM runs; the runner supports `--cpu` (`README.md:123-126`).

Claims:
- **BT gain: supported** as a VM result. The profiler explains it: 90.2 % slot-0 hits, 96 % any-slot hits
  (`docs/profiler/results.md:83-97`). BT's hot helpers (5x5 matrix leaf routines) are exactly the leaf functions the
  filter covers.
- **"Neutral on applications": supported.** It follows from the leaf-only rule: the profiler shows stock-hit rates of
  66.7 % (Redis) and 71.6 % (SQLite) (`profiler/results.md:17-18`), yet no gain, so the hits sit in functions with
  calls.
- **MySQL -2.56 % TPS** with MADs of 0.22 and 0.10 is probably a real small loss. The docs present it as "within 3 % of
  stock" (`results.md:78-80`).

### A4. Code quality and upstream readiness

- Small and self-contained: 259 lines, flag off by default (`-mllvm -tsan-shadow-resident-filter`, `:88-90`), one
  IR test (`llvm/test/Instrumentation/ThreadSanitizer/shadow-resident.ll`) and one runtime test.
- **Portable, which our N1 is not.** Our N1 hard-codes x86-64 Linux (8345a pass: `InlineHitTestTarget` requires
  x86_64 and Linux). This branch exports the mapping constants and so works on AArch64 (several VMA layouts), Darwin and
  any 64-bit little-endian mapping. That is the shape an upstream N1 needs.
- Weak points:
  - an out-of-line call per function entry, paid even when no access executes;
  - the miss path calls the full `__tsan_readN`, which repeats the 4-slot test. Our N1 enters past it
    (`*_miss2`, `MemoryAccessGivenTest`);
  - no layout check; exported ABI globals; leaf-only.
- **Rebase onto 00a2b7021c23:**
  - Runtime: clean or near-clean. Since the base, upstream has 0 commits in `tsan_rtl_access.cpp`,
    `tsan_platform_posix.cpp`, `tsan_interface.cpp` and `tsan_shadow.h`; 1 in `tsan_platform.h`; 3-4 in
    `tsan_rtl.{h,cpp}` (report locking).
  - Pass: a manual port. Upstream moved the TSan `cl::opt`s to TableGen `InstrumentationOptions` (96f2725811e0, 10 Oct;
    372a4c359910, 16 Oct) and changed the `IRBuilder` / `InstrumentationIRBuilder` constructors (ecc201f1d0d5,
    b87aa6ed3b3a), which the branch uses (`:715`).

### A5. How it differs from our N1 (C1, published in ATC'26)

The mechanism is the same: N1 is "inline hit test in the program, runtime only on a miss", EXACT
(runtime/ideas.md row C1). **Nothing in it is new relative to the ATC'26 paper.** The differences are engineering:

| | our N1 (8345a) | shadow-resident |
|---|---|---|
| where emitted | every sanitized function (optionally limited by hot list or loop rule) | leaf functions only |
| fast state | TLS placeholder → `current` → fast_state, inline per access (`loadThr`) | exported call once per function entry, then a load per access |
| shadow mapping | x86-64 Linux constants in the pass, checked at start-up | exported runtime globals (portable) |
| slot order | single test of all 4 slots | slot 0 first, slots 1-3 only on a miss (the staged order fixed the case where the repeated word sits in a later slot, `implementation.md:34-36`) |
| miss path | `*_miss2` (skips the re-test) | full `__tsan_*N` (re-tests) |
| ABI/layout check | `CheckInlineHitTestAbi` at every start | none |
| measured | FFmpeg +5.0 %, Redis +2.5 %, MySQL -1.7 % (x86 `f`) | BT -30 % (VM); apps ±1 %, MySQL -2.6 % |

New relative to our inventory:
- the portable mapping export, usable for an AArch64/Darwin form of N1 (and of N1-ATOMIC C4, N1-ST C3);
- evidence for the staged probe order;
- an AArch64 data point.

---

## B. Parallel note (`feat/parallel-note`)

### B1. Mechanism

- **What is stored:** one u32 "note" per 8 application bytes.
  - It lives in a fixed no-reserve mapping after the meta shadow (`tsan_platform.h:1058-1099`), covering LoApp plus the
    heap range (or HiApp when there is no heap range).
  - The mapping is created by `InitializeShadowMemory` under `one_word_filter=1` (`tsan_platform_posix.cpp:74-102`) and
    re-mapped at `DoReset` (`tsan_rtl.cpp:244-251`).
  - 0 means "shared" (use ordinary shadow); any other value is one TSan shadow word.
- **When it is set:** only by `MemoryRangeImitateWrite`, i.e. malloc. The ordinary shadow of the block is set to
  `kEmpty` and the note to the allocator's write record W(sid, e_alloc, 0xff) (`tsan_rtl_access.cpp:781-787`).
- **Runtime access** (`MemoryAccess`, `:543-572`):
  - Any non-read type (writes, atomics, free/check-only) hands the note over: it is stored into ordinary
    `shadow_mem[0]` and CAS'd to 0 (`HandOver`, `:460-468`, `:552-554`).
  - A read tries `TryExactMerge` (`:441-458`): same sid, and either domination (mask ⊆, epoch ≥, type no weaker) or the
    same epoch and type with a mask union.
  - The note always starts as a write. A read can never dominate it, so in practice the only success is "read in the
    allocation epoch, covered by the write", which returns without trace or store (`:561-562`). Everything else hands
    over.
- **Range, 16-byte and unaligned paths** hand over first (`HandOverRange` at `:596, :634, :868`).
  - `MemoryResetRange` clears the note (`:729`).
  - `MemoryRangeFreed` hands over the first 1 KB and **clears the rest** (`:744-750`).
- **Inline path** (`ThreadSanitizer.cpp:740-797`), in functions with no non-intrinsic calls:
  - checks the region bounds, loads the note and skips the call if `note == cur` (writes) or `note|R == cur` (reads);
  - the fast state comes from `__tsan_one_word_fast_state()` at function entry;
  - atomics and mem-intrinsics are allowed in those functions, unlike in A; this is fine because the fast state is
    re-read per access.
- **Sync, epochs, slots:** the note stores sid and epoch. Any release changes the thread's epoch, so a later access no
  longer matches and the note is handed over. Reset re-maps the notes to 0.

### B2. Soundness

The design argument "never forget history; hand over before it is needed" is right in outline. On reading the code I
found:

1. **Large free drops history stock keeps (PREMISE no-UAF).**
   - Stock `MemoryRangeFreed` marks only the first 1 KB and leaves the shadow beyond it intact (base
     `tsan_rtl_access.cpp:610-614`).
   - The note version clears the notes beyond 1 KB **without handing them over** (`parallel-note:...tsan_rtl_access.cpp:749`),
     while the ordinary shadow there is `kEmpty` (`:786`).
   - Scenario: T1 mallocs 4 KB and frees it. T2, unsynchronised with T1, writes offset 2048 before reuse. Stock reports
     a race with T1's malloc write; the note build reports nothing.
   - The loss lies outside our bar unless no-UAF is invoked. A one-line fix: hand over the whole range, or clear without
     handing over only after the stock path has also reset that shadow.
2. **Stale write in `HandOver`.** It stores the note value into slot 0 *before* its CAS (`:465-466`). Scenario:
   - thread B loads note X and is preempted;
   - thread D hands X over and CheckRaces runs;
   - thread F replaces slot 0 with its own record;
   - B resumes and writes the stale X over F's record. Its CAS then fails.

   F's record is lost. This is a concurrent same-cell window of the kind stock's racy shadow stores also have, but it is
   new and wider. Fix: CAS the note first, then store into the first empty slot, or CAS slot 0 from `kEmpty`.
3. **Darwin-only behaviour change.** With `ignore_interceptors_accesses`, `OnUserAlloc` now imitates a write where stock
   resets the range (`tsan_mman.cpp:285-289`). This may add reports stock does not make; it is not a loss.
4. **DoReset ordering.** The shadow is cleared before the notes are re-mapped (`tsan_rtl.cpp:241-251`). A concurrent
   hand-over in between could plant a pre-reset word in post-reset shadow. That is a possible false positive, not a
   loss. The argument is owed.
5. Read absorption without a trace event changes later eviction choices (`trace_pos`-based victim): class EVICT.
   Reported pairs are unchanged, because the covering W sits in slot 0 in stock as well.

The runtime test (`compiler-rt/test/tsan/one_word_filter.cpp:1-112`) covers state transitions and one race. It has no
negative control for items 1-2, and no test of a large free, ranges, 16-byte or unaligned accesses.

### B3. Results

The note is slower and larger everywhere except the private micro-loop:

| workload | change |
|---|---|
| private loop | -27 % |
| Redis | -31 % throughput, +43 % RSS |
| SQLite | +72 % time |
| BT-W | +253 % |
| IS / CG | +46…64 % |
| FFmpeg | +17…+75 % |
| memcached | -10.75 % throughput |
| MySQL | -23.6 % TPS |

Sources: `results.md:12-16, 44-46, 63-67, 91-110, 128-141`.

The docs draw the right conclusion: not a performance improvement (`results.md:18-20, 48-50`). The cause is a second
mapping touched on every heap access, even after hand-over, since a note of 0 is still loaded. The profiler bounds the
upside: "exact-note beyond CSA" is 0.02-1.44 % of accesses (`profiler/results.md:17-18, 61-67`). That is a clean
**negative result**, and it agrees with our own-heap and CLONE-ESC verdicts: own unpublished blocks are ≤ 0.76 % of
checks (ideas.md:94).

### B4. Upstream readiness

None, and it should not be pursued for upstream. It is invasive: new platform region, allocator hooks, the free path,
reset, and a second ABI.

---

## C. Workload profiler (`feat/workload-profiler`)

- **What it adds:** a runtime census (`tsan_ownership_profile.cpp`, 736 lines, flags at `tsan_flags.inc:40-47`):
  - per-cell owner and private-phase counts;
  - simulated note absorption;
  - stock hit rate by slot (slot 0 vs later);
  - shadow-page density buckets and smaps RSS.
- **Hooks:** behind a global flag in `MemoryAccess`, `MemoryAccess16`, unaligned, range, restart, alloc, free and thread
  start. With the flag off it costs one load and branch per access on the stock path.
- **Branch base:** it is stacked on the parallel-note commits, so it cannot be taken alone.
- **Tests:** `ownership_profile.cpp` and `ownership_profile_density.cpp` check known ratios, which is a good practice.
- **Detection:** unchanged; the profiler always continues into normal `MemoryAccess`.

---

## D. Relation to our work (runtime/ideas.md rows)

| contributor item | our rows | relation |
|---|---|---|
| shadow-resident | **C1 N1** (same mechanism), C2 N1b (miss-path cost), C3 N1-ST, C4 N1-ATOMIC (not covered), G7 (unaligned, neither covers) | duplicate of published N1 with a narrower scope; adds portability (AArch64/Darwin mapping export) and the staged slot order |
| parallel note | **C14 ALLOC-COVER** (its only effective case: the allocator's W covers same-epoch reads), C16 ownership marker (separate per-cell word), R17 / G4 (fresh-memory shadow: the bigalloc RSS win, `profiler/results.md:50-54`), CLONE-ESC / OWN-HANDOFF censuses (ideas.md:94, :153), SUBS C13 (its merge rule = cover lemma, EXACT-G) | a negative result; quantifies C14's read half: 0.65 % (Redis) and 1.44 % (SQLite) beyond CSA, below our est. +1…+3 % |
| profiler | README phase **P1** (where the runtime spends its time) and the slot/hit censuses (R18, R22) | reusable census tool: slot-0 vs later-slot hits, private-phase share, page density |
| archived "fifth shadow slot" (`docs/archive/report/fifth_shadow_slot.md`) | **R23** (eviction victim policy), **R24** (wider shadow), phase **P4** | a victim word per cell, checked only on the miss path: STRONGER than stock (fewer eviction losses). Design only, never built. The best idea in the folder for our P4 detection claim |
| archived "soundness limit" (`docs/archive/report/soundness_limit.md`) | SUBS criterion G, conservatism-review | cover lemma plus a lower bound ("no small per-cell summary is exact"); sound and useful as the motivation for why ownership filters fail |

**Genuinely new relative to our paper and inventory:** the portable N1 form, the staged probe evidence, the
quantified failure of an exact one-word ownership note (with its cover lemma), and the fifth-slot victim design.
**Not new:** the shadow-resident mechanism (= N1, published).

**Where it fits.**
- **Runtime paper (README outline):** the profiler output belongs in P1's motivation (hit rate by slot, private phase,
  page density).
- **A portable N1:** as the upstream form of the ATC inline test, or a short section of the runtime paper.
- **Fifth slot:** as part of claim 4 (fewer races lost than stock), together with R23 and R24.
- **Parallel note:** one paragraph as a negative result.
- **Not a paper on its own.** The positive result is N1 again.

## E. Recommended next steps for the contributor (priority order)

1. **Lift the leaf-only restriction of shadow-resident.**
   - Load the fast state per access from the TLS `current` pointer, as our N1 does, instead of an entry call.
   - Add `*_miss` entries that skip the re-test.
   - Add a start-up layout/ABI check like `CheckInlineHitTestAbi`.

   Goal: one portable N1 that we can propose upstream (x86-64 and AArch64). Rebase onto 00a2b7021c23 and port the
   option to `InstrumentationOptions`.
2. **Negative tests with controls for shadow-resident.** A skip must fire (counter or `__tsan_*` interposer), and each
   near-hit must miss:
   - different mask or offset;
   - read after a narrower write;
   - the same access after `pthread_mutex_unlock`;
   - after `free`;
   - inside `__tsan_ignore_thread_begin`.

   Each case needs a deliberately broken control build that fails it.
3. **Fix parallel-note items B2.1 and B2.2.** Then archive the branch as a negative result with the cover lemma. Stop
   timing it.
4. **Build the fifth-slot victim word** as a flag-gated runtime-only change (R23/R24 territory), with a race-loss
   reproducer whose stock arm loses the race by eviction.
5. **Methodology:**
   - A/A on every host;
   - CPU pinning in the VM;
   - CIs for every row;
   - name the host CPUs;
   - move the claims to native hardware.

**What our lane (tsan-rt) can measure for them:**
- (a) Their shadow-resident build vs stock and vs our N1 on apollo node0: SQLite, FFmpeg, memcached, Redis, MySQL
  Release; A/A, interleaved, 2 layouts; check-tsan on x86-64; race preservation N = 10.
- (b) Their profiler on our 5 apps. It gives P1's slot-hit breakdown and C14's real mass at no implementation cost. It
  needs a rebase without the parallel-note commits.
- (c) Once built, the fifth-slot detection effect with our P4 race-loss benchmark (reproducers and per-site counts).
