# ALS: skipping the no-op join of an acquire-order atomic load

10 Oct 2026. Status: parked by the user's ruling of 10 Oct (the runtime is not to be touched). The code below was
built, audited and timed on 10 Oct before that ruling, against the rule of 25/28 Sep; it is kept as an idea with
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

`../audit-a84-atomic-load-skip.md`: SOUND WITH CONDITIONS at first (a hit skipped the slot lock, where a thread
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

Branch `experiment/als` c911b1f1ff67 (on 8345a0396fa5, runtime only); frozen root
`/extra/alexey/builds/tsan-als-c911b1f1ff67` (not for the paper); census and legs under
`/extra/alexey/chromium/als1`, `als2`, `als2nf`, `als3`.
