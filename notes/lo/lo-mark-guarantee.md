# LO-MARK: what it guarantees, on which premises, and what a void run means

Status 12 Oct 2026. Root `tsan-lomark-913cae99c263`; v1 = spec `sqlite3-lomark5-unheld-c` (roots MemPage, BtShared);
v2 = spec `sqlite3-lomark8-unheld-c-cur` (v1 plus BtCursor). Source of the premise list: audit A79 Q6 (ii) for v1,
audit A81 Q8 for what v2 adds, and the final audit A88 (12 Oct, verdict "quote with changes"), whose corrections
this revision carries (`audits/`). All audits were by reading; the run-time record is in the last section.

## The guarantee

A spec names object types whose fields and buffers are, by the program's design, accessed only under one lock of
the object. The compiler does not prove the lock held: a check is elided only behind a run-time test that the
shadow of the object (or of the buffer pointer used) carries the marker of a lock the thread holds, with no release
possible between that test and the access. The test is an inline compare of one shadow cell with the thread's
held-lock word; a miss can be answered "yes" only by the runtime's mark call, made by a holder. One answer is reused
for later accesses through the same root value while no path in between has a release, an unknown synchronization
or a call that may release. Marked bytes carry the marker instead of access records. Every remaining (checked)
access that meets a marker is
judged at run time: it passes only if the thread holds the named lock or is ordered after the lock's retirement or
the object's release; otherwise the run is void. So for a run that ends without a void:

- every access to marked bytes was made under the named lock or ordered after its release, hence no data race
  existed on those bytes in this execution, and the skipped checks could not have reported one;
- every access to other bytes was checked exactly as the baseline checks it.

A run therefore either reports what the baseline would report on the executed interleaving or is declared void:
under the premises below (9-16 and M1-M3 in particular), and up to the eviction-class differences of premise 11
and M2, it does not silently accept an access that breaks the lock discipline. Without those premises the
sentence does not hold. Two precisions: the tested shadow is the root's first marked granule, or the granule of the
buffer pointer used, not that of every accessed byte (premise 14 carries the rest); and the void case is "a
checked access to marked bytes by a thread that does not hold the lock".

## What "void" means for the user

The runtime prints `ThreadSanitizer: LO-MARK: <cause>; the run is void` and the process exits with code 67. A void
run says the spec's lock discipline did not hold on this execution, or a run-time-checked premise failed. It is not
a race report and not a clean result: the run proves nothing, and its time is not a measurement. The user fixes the
spec (or the program) or reruns that workload with the baseline. A void can occur where the baseline would report a
race, and also where it would report nothing: an unlocked access that is ordered by other means (another lock,
thread start and join) voids as well. A void is therefore not rare by construction, and LO-MARK is not a drop-in
mode for a program whose discipline differs from the spec. (Cosmetic: the first line a void prints is the shared
"EVCONF run void (exit 67)" line.)

## Premises

Checked at run time (a violation voids the run, or the object is refused and stays fully checked):
1. Every access to marked bytes holds the named lock or is ordered after its retirement or release.
2. Locks follow their protocol: no unlock by a non-owner, no destroy under a holder, no destroy unordered with
   the last release, no lock taken while synchronization is ignored.
3. A marked root's pointer and owner members are stored only under its lock.
4. Each marked range is whole granules inside one live heap block of the default allocator; one root per range,
   no root inside another range; at most 8191 lock keys.
5. A buffer's length changes, or a range leaving its block, are detected.

Checked at compile or generation time (the build stops):
6. Every store to a root's buffer pointer or owner member is followed by its re-mark; the stores are listed by
   source position, and a final-IR gate confirms it per build (a condition of every quoted build). For a buffer
   access behind the root's test, that the buffer bytes carry the marker is not tested at the access: it follows
   from this premise and from premise 14. Assumed, not checked: no store exists that has no type tag, no typed
   base and no listed place (A79 item 7).
7. The unit is built with optimization, strict aliasing and debug info. (The flags are checked; that the program
   obeys strict aliasing is assumed.)
8a. No root type is embedded by value or used as an array element (checked by the generator).

Assumed (not checked):
8b. The untyped writers of a root touch only its first 8 bytes: checked for the four known writers, and the list
    of writers is assumed complete (A79 item 16).
9. No use after free, including an access racing a free. This is wider than the ledger's adopted wording ("no
   access through a dangling pointer to freed or reused memory"); the widening is pending a decision
   (`soundness-fixes.md` section 6b).
10. No lock call races the free of the mutex's own memory.
11. Bounded detector state, nearly as in the baseline: a race is lost if a reset or an eviction erased its record,
    or if a release stamp's thread slot was reused by a thread that then passes the ordering test. Not exactly as
    the baseline: see M2.
12. All stores of premise 6 are in units compiled with the spec; none is in inline assembly.
13. Debug locations are faithful.
14. A pointer derived from a tested buffer pointer stays inside the buffer (the declared length is the true bound).
    Known exception: on a corrupt database file SQLite may read up to 16 bytes past a page (A75 L1).
15. An indirect call reaches only functions of its own type; no caller outside the program.
16. The x86-64 Linux shadow layout.

Premises the first version of this note lacked (A88):
- M1. The pass is correct. "Elided only behind a test, with no release possible in between" is a property of the
  compiler, established by reading (A78, A78b, A79) and by the IR test suites. The final-IR gate checks premise 6
  only; nothing checks on the built unit that every unchecked access is dominated by its test. Not examined: the
  reuse of an answer across a longjmp (SQLite's library has none).
- M2. Three record cells beside a retired marker, in v1 too: a destroyed lock's marker stays until a whole-granule
  write or a free, and records are kept beside it in three cells instead of four (1-2 M such stores per run). An
  eviction-class difference from the baseline.
- M3. The other optimizations of the timed arm. The arm is `tsan-dom_peeling-ea-lo-st-swmr` plus the spec; an access
  another analysis removes statically never meets a marker. "Void, never silent" covers LO-MARK's elisions only;
  the arm's figure also rests on the premises of those analyses (`soundness-fixes.md` section 6). No audit examines
  the composition as such.
- M4. The lock filter's premises PM (SQLite's configuration is final once the driver's `main` has set it) and A12
  (the listed units are the whole program). A wrong drop there only elides less.

## What "no annotation" means here

The program is not annotated and no assertion is read. Given by hand or built into the tools: the mutex API's
names (`sqlite3_mutex_enter/leave/try/free/held/notheld`); the type set to cover (`MemPage`, `BtShared`);
`sqlite3_config` by name, with its meaning (premise PM; lockcfg3.py 478-518); the list of units that form the
closed program (`threadtest3`, `test_multiplex`; premise A12); and the candidate rules themselves, which encode
SQLite's locking design as roles (sharable flag, lock mask, link, `locked`/`wantToLock`): the field names are
derived, the roles are not, and the generator has been applied to SQLite alone. These choose candidates only, so
an error costs elisions or a void, not a race. The site list is keyed by source line and column: any edit of the
source needs a regeneration (seen twice: the first seed patch and a regenerated amalgamation were refused).

v2 adds: the release-stamp slot clause of 11 for every explicit release; an object in memory reused without a free
ends its marked life only by a release (a miss voids); the granule sets hold for the closed world of the built
units (an outside unlocked access voids); a granule in release form keeps three record cells instead of four until
overwritten, an eviction-class difference from the baseline. v2's pass and gate conditions (A81 C1, C2) are not
recorded as closed. v1's audit chain is A75, A77-A79, A81 (i) for the multi-lock change with v1's spec, and A88,
which read the three runtime commits made after those audits (ef600f74b644, f524db2e1f25, cc530c49fb6d: a census
flag off by default, a node kept listed, per-thread counters; none changes what the guard answers, marks or voids).

The choice of fields by the spec generator is not trusted: a field that enters the marked set by a wrong guess is
either still checked or, when a thread without the lock touches it, voids the run (within the premises above, for
a build that passes the final-IR gate).

## Record on this root

- Final-IR gate on the leg trees' own compile lines, 12 Oct (preservation host, `wt-dev-cct2/lmpost9/`): v1 PASS
  (6 listed sites, 88 stores in 16 functions each with its re-mark, 94 re-mark calls, 0 release calls); v2 PASS
  (7 sites, 90 stores in 17 functions, 98 re-mark calls, 2 release calls after the 2 named unlink sites); the IR
  with one re-mark removed fails the gate in both arms; only sqlite3.c defines a root type. The pass commit made
  after the last pass audit concerns release lines only (v2); v1's build has none.

- 9 Oct: full preservation (15 threadtest3 tests), N=10 and N=20 runs per arm against stock: no site lost, 0 voids.
  In the N=20 session v1 was below both stock arms at all three WAL sites (stock, stock', v1 of 20: walRestartHdr
  68035 13, 6, 5; 68036 10, 4, 2; walIndexRecover 67451 10, 8, 4); the flag raised then was withdrawn as noise.
- 12 Oct, preservation on this root's leg trees (`lmpres9/` on the preservation host, copy on the build host under
  `/extra/alexey/wt-dev2-r/from-p13/`), 10 runs per arm: no site lost, 0 voids in 20 LO-MARK runs, marks per run at
  least 12,280 (v1) and 425,359 (v2). Runs reporting, stock | v1 | v2: walRestartHdr 68035 8 | 4 | 1; 68036
  6 | 2 | 1; walIndexRecover 67451 3 | 4 | 4; one key undetermined in v2 (3/10 against 0/10).
- 12 Oct, seeded control (`/extra/alexey/wt-dev2-r/lmseed/` on the preservation host): one unlocked write to a page
  of a sharable b-tree from a second thread. Stock reports the race; v1 and v2 void at that write ("a marked
  granule is written without its lock", exit 67); the same binaries without the seed run void-free with the
  elision active (v1 about 1.9 M held hits per 60 s, v2 about 7.9 M); 5 seeded and 2 unseeded runs per arm, built
  from the leg trees' own sqlite3.c and flag lines. Counts: see `seed.log` there.
- 12 Oct, report-rate check (`wt-dev2-r/lmaa/aa.txt` on the preservation host): five arms in parallel, 20 runs each,
  15 tests: stock three times, v1, and the generated spec's arm. No site lost, 0 voids. The timing-dependent WAL
  races are reported by 3 to 10 of 20 stock runs depending on the stock arm (walRestartHdr 68035: 10, 4, 10;
  68036: 10, 3, 5); v1 gives 4 and 5 of 20, inside that spread. Stock against stock also yields one site seen
  only in one arm (3/20 against 0/20).
- Reading of the three sessions together (A88): no difference is resolved. Stock's own arms differ as much as v1
  differs from stock (13 against 4 of 20), so runs are not independent draws and pooled tests are not valid; v1 is
  at the low end of stock's range in both N=20 sessions (lowest or tied lowest in four of six rows). These accesses
  are on the WAL index, outside the marked types, with identical instrumentation in both arms, so a lower rate
  would be a timing effect. "No site lost" is the tool's rule: three sites that stock shows in at most 3 of 20
  runs (67460, 7164, walTryBeginRead 69041) were never shown by v1 in 20; they are undetermined, not kept.
- Limits of the seeded control (A88): one site, one kind (a write), 5 runs; it shows detection on a marked granule,
  not that the access stock pairs the seed with is one whose check v1 elides, and not the other loss paths (a
  stale answer across a release, a missing re-mark), which rest on `lo_mark_rt.c` and the reproducer suites.
- A first version of the seed that shifted source lines was refused at compile time by premise 6's site list.
