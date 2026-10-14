# A95: paper Section 4.2 (SWMR) against the camera-ready implementation

Read-only, 14 Oct. Text: `3a-2-swmr.tex`, `example-prog.tex`, `appendix/lo-proof.tex` at origin/main.
Code: `llvm/lib/Analysis/SingleThreaded.cpp` (`findSWMRGlobals`) and the pass's use site, `ThreadSanitizer.cpp:5626-5637` at
8345a0396fa5 (`:5971` at e52d8787b59e), source `/extra/alexey/wt-dev2-n1cse/c18`. `SingleThreaded.cpp`/`.h` are
identical in all five roots (8345a, e52d, 913cae, b54ce, 429bd); the use site is the same in each. Lines below: SingleThreaded.cpp.

1. "all writes in single-threaded mode" / Prop. "before the program creates further threads": NEEDS QUALIFICATION.
   The base rule matches: a store counts as harmless only if `isSingleThreaded(I)` (2057-2060; in main, a block before
   any thread-creating call, 1096-1116). But `-tsan-swmr-thread-roots` (in memcached's SWMR arm m4W and AllOpt;
   experiments.md:4107) also accepts writes made after threads exist: by main, or by a function that only main calls
   directly, if no thread alive at the write can reach any function that accesses the global (1877-1968). This only works in whole-program
   compiles (1885-1888). This rule gives +0.9 of memcached's +1.4 % SWMR gain (optimization-results.md:218). The pass also
   skips stores to an SWMR global, not only loads (ThreadSanitizer.cpp:5628). So with thread roots, writes from the
   multi-threaded phase go unchecked too. Fix: after the box, add "With thread roots, a write may also follow thread creation if no thread then alive can reach an access to $L$."
2. "sound pointer analysis ... overapproximates the accessed locations": NEEDS QUALIFICATION (misleading). The code has no
   pointer analysis. It applies only to non-constant global variables: it scans the uses of each global's address
   (341-367) and rejects the global if the address goes anywhere other than a direct load or store or an arguments-only
   library call (2088-2095). That rejects non-constant GEPs, stored addresses, other calls, phi and select
   (2032, 2045-2095). Writes through other pointers cannot exist then, so the result is sound but syntactic. The appendix
   footnote "allocation-site abstraction to dynamic allocation" (lo-proof.tex:25-28) describes nothing that exists.
   Fix: "on top of a sound pointer analysis" -> "on a sound over-approximation of accessed locations (in our implementation:
   global variables whose address does not escape)"; drop the footnote.
3. Example, read (3) of g_config elided: NEEDS QUALIFICATION. `int g_config;` and `void initialize_system()` have external
   linkage (example-prog.tex:10, 20). In a compile of one unit at a time, such a global is skipped (2016, 245-254) and
   such a function is multi-threaded (1210-1220), so (3), and (1) by STC too, stay checked. Only the whole-program
   summary step elides them (the -wp lines: Redis, memcached), and that step needs whole-program IR. Fix: declare both `static` in the figure, or
   add "(with whole-program summaries)".
4. Heap/escaping objects, address-taken globals, external linkage: 4.2 states none explicitly. The pointer-analysis framing
   and the appendix footnote suggest heap coverage: FALSE (globals only). Address-taken globals: rejected. Externally
   visible globals: rejected per unit; whole-program summaries only, minus `-tsan-external-symbols` (245-254, 2259-2261).
   3c-pipeline.tex:20 "LO and SWMR focus on global variables": TRUE. Adjacent, outside 4.2: 3c-pipeline.tex:211-215
   says summaries come from per-module analysis "without materializing a whole-program IR". FALSE. Only
   the whole-program pass on linked IR writes them (1297-1299, 1350-1355, 51-57).
5. Measured effect: 4.2 claims none. Removed checks, AllOpt / single arm: memcached 239 / 185, Redis 60 / 48,
   SQLite/FFmpeg/MySQL 0 (experiments.md:4489-4508). Speedups 1.00, 1.00, 0.99, 1.00, 1.01 (paper-results.md:35-41). Claims more:
   (a) 4-experiments.tex:192 "Other analyses eliminate instrumentation ... read-only global variables": NEEDS QUALIFICATION.
   Fix: "(read-only globals: Redis and memcached only)". (b) Stale figures (marked) still show SWMR at MySQL 1.09/1.03 and
   Chromium 1.03/1.04 (4-plots-speedup.tex:182, 204, 219; appendix/chromium-selected.tex:79). Camera-ready values are MySQL 1.000/0.998.
   Chromium has no whole-program step, so SWMR there is the same as stock: these bars must become 1.00.
   Intro 1-introduction.tex:108-110 and 3bb-transformations.tex:112 ("SWMR relies on STC", 2059): TRUE apart from (1).
