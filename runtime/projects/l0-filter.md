# Project 2: Compiler-inlined thread-local memoization ("L0 filter"): audit

State 26 Oct 2026. Read-only audit: nothing was built or run.

- Branch `l0-filter fork: L0Filter` in `/extra/alexey/tsan-rt/repo`: 2 commits on base 4d87e55d8a41 (upstream, 1 Jul 2026):
  - b5f71ea4aab3 "Add initial changes" (6 Jul);
  - d7a513c92648 "Add epoch and handle synchronization" (26 Aug).
- Size: 10 files, +1178/-714. Ignoring whitespace it is +568/-104; the rest is clang-format churn.
- Full diff: `l0.diff` in this folder. The branch's interceptor file is saved as `l0-interceptors.cpp`.
- Citations are `L0Filter:<path>:<line>`, with line numbers in the branch's file.
- No documentation and no result files exist beyond `run.sh`.

## 1. Mechanism

**What is cached, and where:**
- Per OS thread, an initial-exec TLS array `TSanL0Filter[4096]` of u64 and a TLS counter `TSanLocalEpoch`, which starts
  at 1.
- Both are defined in the runtime (`compiler-rt/lib/tsan/rtl/tsan_l0_filter.cpp:15-19`) and again as `weak_odr` in every
  instrumented module (`llvm/lib/Transforms/Instrumentation/ThreadSanitizer.cpp:` initialize, `TSanL0Filter` /
  `TSanLocalEpoch` globals).
- Entry layout:
  - index = (addr >> 3) & 0xFFF;
  - tag = addr & 0x03FFFFFFFFFF8000 (bits 15-57);
  - low 15 bits = local epoch;
  - bit 63 = write.

  Index plus tag identify the **8-byte granule**. The byte offset and the access size are not stored
  (`ThreadSanitizer.cpp:623-697`).

**Hit test, inlined before every instrumented load and store** (`:769`, applied to *every* `OnAccessFunc`: aligned,
unaligned, 16-byte, volatile, compound):
- write hit: `entry == tag|epoch|W`;
- read hit: `(entry ^ (tag|epoch)) & ~W == 0`, i.e. any earlier read or write of the granule in this local epoch;
- on a miss: call the normal `__tsan_*` entry, then store `tag|epoch|W?` into the slot (`:691-697`).
- The array is direct-mapped: a collision only causes a miss.

**Invalidation is the local epoch.** The epoch is advanced:
- **Inline**, before every instrumented atomic load, store, RMW or CAS with acquire, release, acq_rel or seq_cst
  ordering, and before non-single-thread fences (`:852, :872, :893, :917, :940`; `isSyncOrdering` `:1008-1018`;
  `emitEpochAdvance` `:971-1006`). On 15-bit roll-over the whole array is cleared (memset of 32 KB).
- **In interceptors**, via `__tsan_l0_advance_epoch()` (`tsan_l0_filter.cpp:25-33`):
  - `__cxa_guard` and `pthread_once` acquire/release (`tsan_interceptors_posix.cpp:951, :972`);
  - thread start (`:1076`), `pthread_create` (`:1143`), join, tryjoin and timedjoin (`:1175, :1222, :1236`);
  - `cond_wait` after re-lock (`:1340`);
  - mutex lock, trylock, timedlock, clocklock and unlock (`:1453-1544`);
  - spin lock and unlock (`:1579-1599`);
  - rwlock rd/wr lock, trylock, timedlock and unlock (`:1630-1699`);
  - barrier wait (`:1727`).

**Nothing else invalidates:**
- no hook in `DoReset`, slot preemption or re-attach, `free`/`malloc`, `MemoryResetRange` (munmap, stacks), race
  reports, or ignore begin/end;
- no fiber switch hook, no `fork` hook;
- none for the synchronisation TSan models outside those interceptors: `sem_*`, fd acquire/release on pipes, sockets,
  eventfd and epoll, `__tsan_acquire/__tsan_release`, `Annotate*HappensBefore/After`, the `__tsan_mutex_*` annotations,
  libdispatch on Darwin, and atomics executed by code compiled without L0;
- `__tsan_l0_warmup_thread` is defined (`tsan_l0_filter.cpp:35-39`) and never called.

## 2. Soundness: UNSOUND as built

An L0 skip is safe against stock only when stock's own hit test would also skip that access. That needs four things:

- (i) the same TSan fast state (sid, epoch, ignore bit);
- (ii) an identical record (mask, type);
- (iii) that record still in the shadow;
- (iv) the access lying in one cell.

L0 checks none of these. Concrete losses of races stock reports, each a candidate negative test:

1. **Granularity (mask/offset/size):**
   - T writes byte 0 of a granule, then byte 4: the second write hits and is skipped. T2 concurrently reads byte 4. Stock
     has W(mask 0x10) and reports; L0 has only W(0x01) and does not.
   - The same holds for a read of bytes the earlier write did not cover. Struct fields packed into 8 bytes make this
     common.
2. **Two-cell accesses:**
   - an unaligned access spanning two granules, or a 16-byte access (`__tsan_*16`), is keyed on the first granule only;
   - after any earlier access to that first granule, the whole access is skipped and the second granule is never
     recorded or checked.
3. **Release not seen by L0, so TSan's epoch moves and L0's does not:**
   - T: `x = 1; sem_post(&s); x = 2;` → the second write hits.
   - T2: `sem_wait(&s); x = 3;`
   - Stock reports `x = 2` vs `x = 3` deterministically; L0 never does. T2 is ordered after e1 only, and the record stays
     at e1.
   - The same holds for fd-based hand-off (memcached's worker notification pipe; sockets; eventfd), annotations,
     `__tsan_mutex_*` (abseil, folly, Chromium-style locks), libdispatch, and `force_seq_cst_atomics=1` (relaxed atomics
     then release in the runtime but not in L0).
4. **Ignore regions:**
   - an access inside `__tsan_ignore_thread_begin` or `AnnotateIgnoreReads/WritesBegin` misses, TSan ignores it, and
     the miss path still caches the entry (`:691-697`);
   - after the ignore region ends, the same access hits and is never recorded, so later races are lost.
5. **Use-after-free:**
   - T writes `p->a` then `free(p)` (no epoch advance), then writes `p->a` → hit, skipped;
   - stock reports `heap-use-after-free` (upstream test `free_race2.c` is this pattern with a different first access).
   - Also lost: a reallocation of the same address by another thread, accessed by T without synchronisation.
6. **Global reset (DoReset) and slot preemption:**
   - after a reset the shadow is empty, but T's L0 entries remain valid until its next sync, so T's post-reset accesses
     are never recorded. That is a loss stock does not have: stock records them. It is outside P-RESET, which covers only
     pre-reset history.
   - After preemption, T's accesses stay attributed to the old sid at the old epoch. Slot recycling is a P5-like class,
     but here the effect is extended to the access itself.
   - memcached does 456 resets per run (sid-alias.md).
7. **Fibers:** the cache is per OS thread, while TSan threads are per fiber. Fiber B's access hits fiber A's entry, so
   races between fibers are lost.
8. **Eviction (accepted class):** another thread's accesses can evict T's record. Stock would re-record it on T's next
   access; L0 does not. This is the P-EV class we accepted on 1 Oct. It is the reason our own loop guard DE-LC was
   retired as inexact (ideas.md:122).
9. **Two definitions (Darwin):** the TLS variables are defined `weak_odr` in each module and strongly in the runtime.
   With the static Linux runtime the linker keeps one copy. On Darwin (dylib runtime, two-level namespace) the program
   may bind to its own copy, and then the interceptors' advances are invisible to the inline test. `run.sh` used
   `DYLD_FORCE_FLAT_NAMESPACE=1`, which may have hidden this. Unverified; needs a test.

Item 3 sharpened: on acquire, TSan's own epoch does not move, so advancing L0 at an acquire is only conservative. What
soundness needs is an advance at **every** release TSan performs. The interceptor list misses many of those releases.

**Premises it relies on as built:**
- no unaligned or 16-byte access to a cached granule;
- no sub-granule mixing;
- every release goes through the listed interceptors or through L0-compiled atomics;
- no reset, preemption, ignore region, free or fiber.

Even our least strict accepted class (EVICT plus no-UAF) does not cover items 1-4, 6 and 7.

**A sound form** (what it would take to reach "EVICT only"):
- Tag entries with TSan's **FastState word itself** (sid, epoch, ignore bit) instead of a private epoch. Every release,
  slot change and ignore change then invalidates automatically, with no interceptor hooks.
- Key on the exact **(granule, mask, type)**, or restrict hits to identical size and offset.
- Limit it to aligned 1-8-byte accesses.
- Add a **global generation**, bumped at DoReset and at frees of cached memory (our C18 `__tsan_shadow_generation`
  pattern), compared inline. Alternatively, accept no-UAF for the free part.
- Hook the fiber switch.

What remains is eviction (P-EV), as for DE.

## 3. Results as reported

There are none beyond `run.sh` (`L0Filter:run.sh:1-9`):
- macOS (xcrun, dylib interposer), `-O0`, a two-function micro-benchmark (`test.cpp`);
- calls into `__tsan_read*/write*`: 20,129 → 65.

No timing, no applications, no A/A, no repetitions, no race-detection check. The claim "detects dynamically redundant
accesses" is shown only as a call count, and the count includes the unsound skips. For example, `sum_array` reads after
writes with the same mask, which happens to be safe here; nothing distinguishes the safe skips from the unsafe ones.

## 4. Code quality and upstream readiness

**Not ready:**
- **On by default:** `-tsan-enable-l0` is `cl::init(true)` (`ThreadSanitizer.cpp:53-55`). This changes stock TSan for
  every build and will break the existing `llvm/test/Instrumentation/ThreadSanitizer` FileCheck tests.
- **Does not compile on Linux:** dangling `else` after the inserted advance in `pthread_tryjoin_np` and
  `pthread_timedjoin_np` (`tsan_interceptors_posix.cpp:1220-1223, :1234-1237`, inside `#if SANITIZER_LINUX`). Builds
  were done only on macOS.
- **No tests:** no lit IR test, no compiler-rt test, no negative test. The repository root holds scratch files
  (`L0Filter.cpp`, `run.sh`, `test.cpp`, `wrap_tsan.cpp`, `.gitignore` entries).
- **~700 lines of unrelated clang-format churn** in `tsan_interceptors_posix.cpp` and `tsan_rtl.h`.
- C++ `thread_local` in the runtime (compiler-rt uses `THREADLOCAL`). A 32 KB per-thread static TLS block. The cache is
  as large as an L1D, and its lines track the program's own lines one to one, so L1 pollution is a real risk; it has
  not been measured.
- **Rebase onto 00a2b7021c23:**
  - the pass needs a manual port: options moved to TableGen `InstrumentationOptions` (96f2725811e0, 372a4c359910) and
    `IRBuilder` constructors changed (ecc201f1d0d5, b87aa6ed3b3a);
  - `tsan_interceptors_posix.cpp` has 9 upstream commits since the base, which will collide with the formatting churn;
  - drop the churn first. The functional runtime part (one file plus about 26 one-line calls) is small.

## 5. Relation to our work

| L0 aspect | our rows | relation |
|---|---|---|
| per-thread "already checked this granule since the last sync" | **DE** (static dominance elimination, ATC'26); **DE-LC** loop guard (ideas.md:122: per-site flag, sync counter + `__tsan_shadow_generation`; retired 25 Sep as inexact under eviction, its exact form is N1-L); **DE-AV** dynamic availability (ideas.md:123: a flag per pointer value, cleared at sync; bound FFmpeg ~12 %, SQLite ~6 %, memcached ~7.5 %, Redis ~2.3 % of hits) | L0 is the **address-keyed, run-time** version of DE-AV / DE-LC; it crosses calls and pointer values, which DE cannot |
| skipping the runtime call on a repeat | **C1 N1** (exact; published) | N1 already removes the call for every stock hit, exactly. L0's remaining advantage over N1 is only a denser probe (8 B TLS entry vs a 16 B shadow cell), i.e. cache locality on cold shadow |
| invalidation by generation | **C18** DE generation guard (rejected 25 Sep), LG-2/3/6 (ideas.md:27: reset, report clear, foreign mmap) | the same obligations; L0 meets none of them |
| epoch handling | R20/R21 (epoch increments), R5/R6 RT-SYNC-LF / ALS (version caches) | ALS shows how to make a per-thread cache exact against TSan's clocks (A84 audit); L0 should reuse that discipline |
| report-content effects | K13 P-REPORT | a second race on a reported cell may be lost; same class as DE |
| literature | FastTrack's same-epoch rule; ReX (ATC'26 comparison, `4-experiments.tex:169-178`) | the idea is standard. Novelty would lie only in making it exact against TSan v3's bounded shadow and in showing a gain *on top of* N1 |

**New relative to our inventory:** the address-keyed thread-local cache in front of the runtime, with invalidation at
sync, is not in runtime/ideas.md. Its closest relatives (DE-LC, DE-AV) are site- or value-keyed and live in the compiler
inventory. **Not new:** the skip rule (stock CSA / N1) and the generation-guard obligations (C18).

**Fit:**
- At most a subsection of the runtime paper's fast-path material, as "a thread-local filter in front of the inline
  test", if the sound form measurably beats N1 alone.
- Not a paper on its own: the unsound form cannot be reported, and the sound form's upside is bounded by
  N1-vs-stock-hit costs.

## 6. Recommended next steps for the contributor (priority order)

1. **Make it build and keep stock as the default:**
   - `cl::init(false)`;
   - fix the two dangling `else`s;
   - drop the format churn and the scratch files;
   - rebase onto 00a2b7021c23;
   - run check-tsan and the TSan IR tests on Linux x86-64.
2. **Negative tests with controls, one per item in §2** (granule offset, unaligned/16-byte, `sem_post`, pipe hand-off,
   ignore region, use-after-free, reset via `__tsan_flush_memory` or epoch exhaustion, fibers):
   - each must report under stock and fail under the current L0, proving that the elision fires;
   - each must then pass after the fix.
3. **Redesign to the sound form in §2.** That means FastState-tagged entries, exact (granule, mask, type), aligned 1-8
   bytes only, and a generation counter for reset and free. Then write the soundness note: EXACT except P-EV, plus
   no-UAF if the free part is not guarded. Get an independent audit.
4. **Measure only after 1-3,** against **N1** (not only against stock):
   - census: the share of runtime calls that are same-thread same-granule repeats, split into "stock would hit" and
     "stock would miss" (the second set is the unsound mass);
   - then time L0 alone, N1 alone, and L0+N1.

**What our lane can measure for them:**
- (a) A/A-calibrated legs on apollo node0 for stock / N1 / L0-sound / L0-sound+N1 on our 5 applications. Interleaved
  arms, 2 layouts, 95 % bootstrap intervals.
- (b) Race preservation N = 10 per site, plus our existing loop-guard reproducers in `/extra/alexey/de-lc/evidence`.
  They target exactly the eviction, sync and reset holes an L0 cache shares.
- (c) A cheap census from our access-stats counters (`perf/access-stats` lineage): repeat-hit share per application.
  This tells in advance whether the cache-locality upside over N1 exists.
