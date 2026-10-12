# ALS: skipping the no-op join of an acquire-order atomic load

10 Oct 2026. Status: parked (decided 10 Oct: the runtime is not to be touched). The code below was
built, audited and timed on 10 Oct before that decision, against the rule of 25/28 Sep; it is kept as an idea with
its evidence. Nothing from it is in the paper's tables.

## 1. The cost

Stock TSan, for an atomic load whose order is not relaxed and whose address has a sync object: a walk from the
meta shadow to the sync object, a lock and unlock of the thread's slot mutex, a reader lock on the sync object (a
line shared by all readers of that atomic), a join of its vector clock over 256 slots, a second load of the value,
then the ordinary access check. One thread: 52 ns against 3.9 ns for a plain read and 8 ns for a relaxed load;
eight threads loading one atomic: about 2,000 ns per load.

On Chromium under stock TSan this path is 21-32 % of the browser's CPU samples (parser and svg stories):
`__tsan_atomic*_load` 10-16 %, `MetaMap::GetSync` 6-9 %, slot lock and unlock 4-6 %, the join 1 %.
Stores, read-modify-writes and real mutexes together are under 2 %.

## 2. The mechanism

- A version word in the sync object names the present state of its clock: 0 while a writer is changing it, 1 when
  the clock is empty and nobody is writing, otherwise a value unique in the process and never reused.
- Every writer of a sync clock (atomic store, RMW, CAS; mutex unlock paths; the Release family; reset) sets 0 under
  the write lock before its change can be seen and a fresh version after; a failed CAS puts back what it found.
- A thread caches (address, version) for clocks it has joined on the slow path.
- A load reads the value, then the version; if the version is "empty", or equals the cached one, and the thread
  still owns its slot (stock's own early-return condition of the slot lock), it skips the slot lock, the reader
  lock and the join. Re-joining an unchanged clock, or joining an empty one, changes nothing.
- The version is read before the slot check: a reset clears the slot's owner before it stores "empty".

A first design with a (slot, epoch) stamp and a dominance test was rejected: a thread that attaches to a used slot
does not join the previous user's clock, so the stamp does not imply dominance.

## 3. Audit

Audit A84 (its report is not in this repository): SOUND WITH CONDITIONS at first (a hit skipped the slot lock, where a thread
notices a stolen slot; a race could be lost), SOUND after the slot-ownership check and through three deltas (empty
clock version, read order, cache size flag). Not covered by tests: the window inside a compare-exchange, a reset
racing another thread's load, fibers, fork. A blind spot of the base runtime was noted on the way: a report after a
plain slot re-attach can be dropped because the trace holds no time event for it (by reading, not run).

## 4. Numbers

- Gates at c911b1f1ff67: full suite x12 flag off (392) and with the skip forced on (391), Go build, inert against
  the parent root (same objects), flag-off cost 0.1-0.4 ns on relaxed operations.
- Chromium census, share of with-sync loads eligible: 94-97 % with the harness's `flush_memory_ms=2000`, 86-90 %
  under default options (svg: all through the per-thread cache; parser: 58 % empty clocks even without the reset).
  A 256-entry cache instead of 64 changes nothing: the remaining misses are first contacts.
- Timing, one binary with the flag off and on, 49 blink stories, default options: AMD 1.022, Intel 1.021; parser
  1.13 on both, layout 1.01, the rest inside the A/A. No memory or code cost.

Caveat on the numbers: the census counters are themselves a runtime change, so every ALS census ran on a patched
runtime. The only figures taken on an unmodified runtime are the profiles of section 1 and the stock micro-benchmark.

## 5. Where it lives

Branch `experiment/als` c911b1f1ff67 (on 8345a0396fa5, runtime only); frozen root `tsan-als-c911b1f1ff67` (not for
the paper). The census and timing data are kept on the Intel host, not in this repository.

The relinked browsers were removed from the Intel host's scratch disk on 10 Oct 2026 (the line is parked and the
room was needed for builds): `chrome-cr1n1-als` (chrome sha256 712a33bd626e1830), `chrome-cr1n1-als2`
(5e5e6463cbf65941), `chrome-cr1n1-als-c911b1f1ff67` (121ddcc66e10dede). One relinked pair remains there:
`chrome-cr1r` (CR1 rebuilt, with `obj/`) and `chrome-cr1r-als-c911b1f1ff67` (605e9d99b0ccbc5c). To make one again:
an out dir of the configuration with its `obj/` tree (about an hour's build on the Intel host), then a relink script
taking a tag, the directory of runtime archives, the out dir and a sibling dir: a 15-second link that replaces only
`chrome`; only the executables embed the runtime. After the removal the frozen CR1+N1 browser (121ddcc66e10dede)
exists only as a copy on the second host (`chrome-cr1n1alsf`), and the provisional one of commit 9dbd02c7cfac
(5e5e6463cbf65941) exists nowhere; its runtime archives are kept on the Intel host.
