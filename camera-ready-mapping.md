# Camera-ready: the new optimizations under the paper's analyses

State: 12 Oct 2026. Figures: [`optimization-results.md`](optimization-results.md). Premises:
[`optimization-results.md` §8](optimization-results.md#premises), [`soundness-fixes.md`](soundness-fixes.md) §6.
Terms: [end of file](#terms). Hosts: AMD, except Chromium (Intel).

## 1. Mapping

**In short.** Each new optimization is assigned to one of the paper's analyses and reported under its name.

- **Four of the five analyses (STC, LO, EA, DE) get a variant checked at run time**, as STC has DynSTC in the
  submitted paper. The analysis removes TSan's check where it can prove its condition at compile time (LO: "every
  access to the variable holds one lock"). The run-time variant leaves a short test of the same condition in the
  code ("is the lock held now?") and calls TSan only if the test fails. These are DynSTC-RT, N1-ST and quiet threads
  (STC); LO-OBJ (LO); EVCONF-CHECKED (EA); verified DE and N1 (DE).
- **Four optimizations widen a static rule**: SWMR-ROOTS (SWMR), EA-CONTENTS and MEMINTR (EA), VWIDE-loops (DE).
- **FE-INL** (inlined function entry and exit) belongs to no analysis and gets a paragraph of its own.

Almost all adopted figures come from the run-time variants. For quiet threads, LO-OBJ and EVCONF-CHECKED the
assignment is disputable; the objections are in their entries.

Link: **direct** = the same rule, one condition changed; **idea** = the same property, established differently;
**weak** = a reviewer may object.

| paper | property | merged in: static | merged in: run time |
|---|---|---|---|
| STC, DynSTC | one thread exists | — | [DynSTC-RT](#dynstc-rt), [N1-ST](#n1-st) (direct); [quiet threads](#quiet) (weak) |
| SWMR | every write precedes the threads | [SWMR-ROOTS](#swmr-roots) (direct) | — |
| LO | a held lock owns the location | — | [LO-OBJ](#lo-obj) (weak) |
| EA | the object is confined to one thread | [EA-CONTENTS](#ea-contents) (direct); [MEMINTR](#memintr) (idea) | [EVCONF-CHECKED](#evconf) (weak) |
| DE, peeling | an earlier check of the same location covers this one | [VWIDE-loops](#vwide) (idea) | [verified DE](#verified-de) (direct); [N1](#n1) (idea) |
| — | — | [FE-INL](#fe-inl): maps to nothing; a paragraph of its own | |

What carries each adopted figure:

| app | submitted | adopted | carried by | reported under |
|---|---|---|---|---|
| SQLite | 1.71× | 1.19× | LO-OBJ | LO |
| FFmpeg | 1.57× | 1.29× | DynSTC-RT, N1, N1-ST | DynSTC, DE |
| Redis | 1.45× | 1.54× | quiet threads, N1, FE-INL | DynSTC, DE, FE-INL |
| MySQL | 1.16× | 1.13× | FE-INL | FE-INL |
| memcached | 1.07× | 1.80× | EVCONF-CHECKED | EA |
| Chromium (Intel, hold-out) | 1.39× | 1.25× | N1, FE-INL, MEMINTR | DE, FE-INL, EA |

Quiet threads, EVCONF-CHECKED and LO-OBJ were each built for and measured on one application (Redis, memcached,
SQLite).

## 2. The paper's analyses

| name | rule (paper) | scope |
|---|---|---|
| STC | an access executed only before the first thread is created is not checked (`3a-1-stc.tex:23`) | per function; per block in `main` |
| SWMR | a read is not checked if every write to its locations is in single-threaded context (`3a-2-swmr.tex:25`) | globals |
| LO | an access is not checked if all its locations are owned by held locks (`3a-3-lo.tex:28`) | globals; must-held locksets |
| EA | an access to an object that does not escape its thread is not checked (`3a-4-ea.tex:28`) | stack and heap objects |
| DE | a check dominated by a check of the same location, with no release-like instruction between, is removed (`3b-redundancy.tex:56-66`); peeling splits off a loop's first iteration to create the dominating check | interprocedural paths |
| DynSTC | `if (__thread_cnt > 1)` around each check (`3bb-transformations.tex:31-35`) | every check |

## 3. The new optimizations

Results: change of speed from adding the optimization ([`optimization-results.md` §3](optimization-results.md#single)),
or speedup over stock (×).

### STC / DynSTC

<a name="dynstc-rt"></a>
**DynSTC-RT** (direct). DynSTC's test inside the runtime: while one thread is alive the runtime records nothing.
A runtime option; the binary is unchanged. FFmpeg +12.0 % (the compiled guard: +10.1 %); others ±2 %.
The submitted DynSTC already keeps its counter in the runtime.

<a name="n1-st"></a>
**N1-ST** (direct). DynSTC's guard around the inline test of [N1](#n1). FFmpeg +16.9 % over N1 + DynSTC-RT.

```c
// N1:     if (no_matching_record(p)) __tsan_read4_miss(p);
// N1-ST:  if (multithreaded && no_matching_record(p)) __tsan_read4_miss(p);
```

<a name="quiet"></a>
**Quiet threads** (weak). From "one thread exists" to "one thread is active". The compiler derives from the whole
program where Redis's I/O and background threads park; while each has been parked since a lock release that the
main thread acquired, the main thread skips its checks. Redis +31.9 %.

```c
if (all_other_threads_quiet()) {    // guard, main thread
    reply->len += n;                // unchecked
    memcpy(reply->buf, s, n);       // unchecked
} else { /* the same code with the checks */ }
```

- Link: the paper motivates DynSTC with thread pools whose threads were joined (`3bb-transformations.tex:19-21`);
  here the threads are parked. Parking points come from a static whole-program analysis.
- Against: the threads are alive, so DynSTC's theorem does not apply; the argument uses the happens-before edge
  through the lock, with premises checked for Redis only. Needs a whole-program summary step.

### SWMR

<a name="swmr-roots"></a>
**SWMR-ROOTS** (direct). "Before the threads" counted per thread entry function: reads of a global are unchecked if
every write precedes the creation of each thread that can reach it. memcached +0.9 %; inert on Chromium (needs the
whole program).

```c
settings.port = 11211;              // every write is before the create
pthread_create(&t, 0, worker, 0);
// in worker:  use(settings.port);  // before: __tsan_read4(&settings.port);  after: nothing
```

### LO

<a name="lo-obj"></a>
**LO-OBJ** (weak). LO for heap objects whose lock is a field of the object's owner: a spec names the lock of each
object type, and an access is unchecked while the thread holds that lock (tested at run time). The spec is generated
from the program's assertions (`assert(sqlite3_mutex_held(pBt->mutex))`). SQLite 1.19× (hand-written spec: 1.25×);
static LO finds almost nothing there.

```c
sqlite3_mutex_enter(pBt->mutex);
pBt->nPage++;                       // before: __tsan_write4(&pBt->nPage)
                                    // after:  if (!held(pBt->mutex)) __tsan_write4(&pBt->nPage);
sqlite3_mutex_leave(pBt->mutex);
```

- Against: LO proves ownership; LO-OBJ assumes it (premises R3, P-OWN, B2). The test shows that this thread holds
  the lock, not that every other access does. A program without such assertions gets nothing.
- **LO-MARK**, the variant without assertions: the first access under the lock writes a marker of the lock into the
  object's shadow; a checked access by a thread that does not hold the lock and meets the marker voids the run, so
  the ownership assumption is tested at run time. It still needs, per type, where the owner pointer and the lock
  field are. SQLite, three legs: 1.06×, 1.14×, 1.26×, against 1.10×, 1.17×, 1.23× for the generated spec in the same
  legs (ratio 0.97, 0.97, 1.02). Not adopted: other compiler root, premises not written up, no preservation run.

### EA

<a name="ea-contents"></a>
**EA-CONTENTS** (direct). Publishing a pointer read out of a private container no longer makes the container escape.
memcached +1.4 %.

```c
item *slots[8];                     // private array
publish(slots[i]);                  // the item escapes; the array does not
slots[j] = 0;                       // before: __tsan_write8(&slots[j]);  after: nothing
```

<a name="memintr"></a>
**MEMINTR** (idea). A `memcpy` whose source is a private object (EA's fact) or a constant checks only its
destination. ±2.5 % alone; part of Chromium's FE-INL + MEMINTR step (+9.8 %), its share not separated.

```c
char buf[64];                       // private
memcpy(shared->data, buf, 64);      // before: both ranges checked;  after: only shared->data
```

<a name="evconf"></a>
**EVCONF-CHECKED** (weak). In an event-driven server a connection's buffers are handed to one worker and then used
by that thread only. The analysis proposes such memory; run-time owner checks and a marker in the shadow make each
elision sound; a failed owner check voids the run. memcached 1.80×, no annotations (1.81× with seven annotated
fields, the earlier form EVCONF).

```c
char *p = c->rcurr;                 // c->rcurr itself: checked
n = p[0];                           // before: __tsan_read1(p)
                                    // after:  if (!owned_by_this_thread(p)) __tsan_read1(p);
```

- Link: EA's property, "confined to a single thread" (`3a-4-ea.tex:2`), tested at run time. Of the executed checks
  on memory that is in fact one-thread, the heap reachable from shared structures is 79 % on SQLite, 42 % on Redis,
  34 % on memcached; static escape rules do not reach it.
- Against: these objects do escape in EA's sense; this is ownership transfer.

### DE

<a name="verified-de"></a>
**Verified DE** (direct). DE's covered check is kept as a short inline test that the dominating check's record is
still in the shadow cell; the runtime is called only if it is gone.

```c
x = p->len;                         // __tsan_read4(&p->len)
y = p->len + 1;                     // before: __tsan_read4(&p->len)
                                    // after:  if (record_of_the_check_above_is_gone(&p->len)) __tsan_read4(&p->len);
```

- Purpose: soundness on a detector with bounded shadow, at the speed of removal (within 1 %). With four records per
  cell the dominating record can be displaced; removal then loses a race that stock reports. Removal is sound only
  under premises P-EV, P-RESET and P-REPORT; the verified form needs none of them. This is the content for the
  paper's text on DE and bounded shadow memory.
- Consequences: DE no longer lowers the count of instrumented accesses (the paper's static reduction figures for
  DE change meaning); the test is [N1](#n1)'s, so where N1 is at every check DE adds nothing.

<a name="vwide"></a>
**VWIDE-loops** (idea). The verified form at checks in loops that nothing dominates but where an earlier check of
the same address is likely (static estimate). Over FE-INL: Redis about +2.4 %, MySQL about +1.0 %. Not in the
adopted configurations: Redis has N1 at every check, which includes it; on MySQL the 1 % is inside the noise.

```c
for (i = 0; i < n; i++)
    if (v[i] > 0)
        sum += a->total;            // no iteration's check dominates the next
                                    // after: inline test, runtime call only on a miss
```

<a name="n1"></a>
**N1, inline hit test** (idea). The check compares in place whether the cell already holds the same access by the
same thread, and calls the runtime only on a miss. Placement variants (small functions S56; static frequency s1, s4;
small loops K4) inline the test at fewer checks.

```c
x = p->field;
// before:  __tsan_read4(&p->field);
// after:   if (no record in the cell == this access, this thread, this epoch)
//              __tsan_read4_miss(&p->field);
```

- Link: DE's condition evaluated at run time. DE: an earlier access, same location, at least as strong, no
  release-like instruction between. A matching record: this thread, this location, this kind of access, epoch not
  advanced (TSan advances it at release-like operations). DE proves it for all paths and needs dominance and
  must-alias; N1 observes it on the path taken. The placement variants are the static half.
- Answers the ReX comparison (`4-experiments.tex:177`): a dynamic redundancy filter at near-zero per-access cost.
- Results: Chromium +16.5 % (first sample), +13.4 % (hold-out) over FE-INL + MEMINTR on Intel; +1.2 %, −3.8 % on
  AMD. FFmpeg +5.0 %. On top of the ownership configurations it loses: memcached −5 %, SQLite −18.5 %.
- Cost: 133–140 bytes per check; Chromium code 3.0×, peak memory of the largest process 1.77×; compile time FFmpeg
  +97 %, Chromium +10 %, most of Redis's +168 %. x86-64 Linux, aligned accesses of 1–8 bytes.
- Against: stock's runtime does this comparison as its first step, so it can be called inlining of the fast path;
  it removes no check.
- Open: two server deaths in 104 MySQL runs with it (none in 80 later runs); a lower report rate of one Chromium race;
  two of about 26 Chromium Speedometer runs with it died on a V8 debug check in the garbage collector (no other
  build did; the source points to a timing hazard in V8 itself, under test, see
  [`optimization-results.md` §2](optimization-results.md#chromium)).

### No analysis

<a name="fe-inl"></a>
**FE-INL**. The shadow call stack's push and pop are done in place instead of by runtime calls at every function
entry and exit. MySQL +3.6 % (its whole 1.13× configuration), Redis +4.8 %, Chromium +9.8 % with MEMINTR. Not an
access check; no soundness argument needed (the recorded stack is the same). Place: a paragraph in `sec:trans` or
after the remark "The pipeline can be extended with further analyses" (`3c-pipeline.tex:25`).

```c
__tsan_func_entry(pc);              // after: *thr->shadow_stack_pos++ = pc;
__tsan_func_exit();                 // after: thr->shadow_stack_pos--;
```

Not for the paper (in no adopted figure): FE-SINK, N1-L, k2, the annotated EVCONF variants, Chromium's
library-facts table, the families of [`optimization-results.md` §4](optimization-results.md#variants).

## 4. Changes to the paper

| section | change | size |
|---|---|---|
| Introduction | five static analyses, four with a run-time form | ~15 lines rewritten |
| SWMR | thread roots | ~6 lines |
| EA | contents rule; memory intrinsics | ~5 lines |
| DE (`sec:beyond`) | verified form, bounded shadow memory; N1 as its run-time form, placement | ~0.5 column |
| `sec:trans` | DynSTC: runtime form, N1-ST, quiet threads; run-time LO (LO-OBJ); run-time EA (EVCONF-CHECKED); FE-INL | ~1 column |
| Pipeline | figure, order, summary step | ~8 lines |
| Evaluation | updated figures under the old names; compile time | separate task |

About 1.5–2 columns of new text; the camera-ready version has one extra page.

## 5. Decisions (12 Oct)

1. N1 is part of DE.
2. The new optimizations are merged into the paper's analyses; updated figures are published mostly under the old
   names.
3. The whole-program summary step does not weaken "fully automatic". The use of assertions (LO-OBJ) is to be settled.
4. DynSTC-RT as a runtime option is accepted.
5. Soundness statements for quiet threads, LO-OBJ and EVCONF-CHECKED: to be written and discussed separately; the
   premises are listed in [`optimization-results.md` §8](optimization-results.md#premises).
6. All figures on AMD; Chromium on Intel for now.
7. Peeling stays, as part of DE.
8. Space: one extra page.

Open: LO-OBJ with assertions, or LO-MARK without them.

<a name="terms"></a>
## Terms

| term | meaning |
|---|---|
| shadow cell | TSan keeps four records of recent accesses per 8 bytes of memory |
| epoch | a per-thread counter in each record; advanced when the thread releases a lock or creates a thread |
| voids the run | the run stops with an error naming the failed check; a wrong ownership guess is never a silently lost race |
| summary step | a pass over the whole program before the units are compiled |
| first / hold-out | Chromium's two story samples: 49 used during development / 48 others fixed before they ran |
