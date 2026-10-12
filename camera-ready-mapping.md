# Camera-ready: mapping the new optimizations onto the paper's analyses

State: 12 Oct 2026. Purpose: decide where each optimization behind the camera-ready figures goes in the paper.
Figures and longer explanations: [`optimization-results.md`](optimization-results.md) (sections 1, 3 and 6).
Soundness and premises: [`soundness-fixes.md`](soundness-fixes.md).

The examples are schematic C. `__tsan_read4(p)` / `__tsan_write4(p)` stand for TSan's check of a 4-byte access.

## Contents

1. [The proposal in one table](#proposal)
2. [The paper's optimizations](#old)
3. [The new optimizations, each under the analysis it extends](#new)
4. [What does not map](#nomap)
5. [What the paper would look like](#structure)
6. [Open decisions](#open)

<a name="proposal"></a>
## 1. The proposal in one table

The paper's task list this file serves is the camera-ready list of 12 Oct: item 1 is this mapping, item 4 the
fallback when a mapping fails, item 5 the text on DE and bounded shadow memory, item 6 the page limit.

The organizing idea: **four of the paper's five static analyses get a run-time counterpart** (SWMR gets only a
static refinement), as STC already has DynSTC in the submitted paper. The static analysis proves a property for all executions and removes the check; the
counterpart tests the same property in the current execution and skips the check while it holds. Two further
optimizations are plain refinements of a static rule. One optimization does not map.

Strength of the link: **direct** = the same rule with one condition changed; **by idea** = the same property,
established differently; **stretched** = defensible, but a reviewer may object.

Terms used below. *Shadow cell*: TSan keeps four records of recent accesses per 8 bytes of memory. *Epoch*: a
per-thread counter in each record, which TSan advances when the thread releases a lock or creates a thread.
*Voids the run*: the run stops with an error naming the failed check, so a wrong ownership guess is never a silently
lost race. *Summary step*: a pass over the whole program before the units are compiled. *CR1*: Chromium's
configuration, the four analyses + FE-INL + MEMINTR. *First / hold-out*: Chromium's two story samples (49 stories
used during development / 48 others fixed before they ran). Hosts: AMD unless "Intel" is written.

| paper's optimization | property it relies on | static refinement | run-time counterpart |
|---|---|---|---|
| [STC](#old-stc) / [DynSTC](#old-dynstc) | only one thread exists | — | [DynSTC-RT](#dynstc-rt), [N1-ST](#n1-st) (direct); [quiet threads](#quiet) (stretched) |
| [SWMR](#old-swmr) | every write precedes the threads | [SWMR-ROOTS](#swmr-roots) (direct) | — |
| [LO](#old-lo) | a lock owns the location | — | [LO-OBJ](#lo-obj) (stretched) |
| [EA](#old-ea) | the object is confined to one thread | [EA-CONTENTS](#ea-contents) (direct); [MEMINTR](#memintr) (by idea, partly stretched) | [EVCONF-CHECKED](#evconf) (stretched) |
| [DE](#old-de) | an earlier check of the same location covers this one | [VWIDE-loops](#vwide) (by idea) | [verified DE](#verified-de) (direct); [N1](#n1) (by idea, the boldest claim) |
| — | — | [FE-INL](#fe-inl): does not map | |

Which of them carry the adopted figures (so they must be described in the paper):

| app | submitted paper | adopted figure | carried by |
|---|---|---|---|
| SQLite | 1.71× | 1.19× | [LO-OBJ](#lo-obj) |
| FFmpeg | 1.57× | 1.29× | [DynSTC-RT](#dynstc-rt), [N1](#n1), [N1-ST](#n1-st) |
| Redis | 1.45× | 1.54× | [quiet threads](#quiet), [N1](#n1), [FE-INL](#fe-inl) |
| MySQL | 1.16× | 1.13× | [FE-INL](#fe-inl) |
| memcached | 1.07× | 1.80× | [EVCONF-CHECKED](#evconf) |
| Chromium (Intel, hold-out) | 1.39× | 1.25× | [N1](#n1), [FE-INL](#fe-inl), [MEMINTR](#memintr) |

Three of the run-time counterparts were each built for and measured on one application: quiet threads on Redis,
EVCONF-CHECKED on memcached (with a list of libevent functions), LO-OBJ on SQLite (its spec generator reads SQLite's
assertions). The paper can present them as instances of a pattern, not as general-purpose passes.

<a name="old"></a>
## 2. The paper's optimizations

Quoted from the submitted text; file and line in the paper's repository.

<a name="old-stc"></a>
**STC, single-threaded context** (`3a-1-stc.tex:23`). "Any memory access instruction that is not in a MT function
or basic block is guaranteed to only be executed before any further threads are created, and thus cannot be involved
in any races and need not be instrumented." Call-graph analysis, per function (per basic block in `main`).

```c
int main(void) {
    g_config.n = 4;                 // before: __tsan_write4(&g_config.n);  after: nothing
    pthread_create(&t, 0, worker, 0);
}
```

<a name="old-swmr"></a>
**SWMR, single writer / multiple readers** (`3a-2-swmr.tex:25`). "A memory access instruction need not be
instrumented if all writes to locations potentially accessed by it occur in ST context." Global variables; uses STC.

```c
void *worker(void *a) {
    int n = g_config.n;             // before: __tsan_read4(&g_config.n);   after: nothing
}                                   // (g_config.n is written only in main, before the threads)
```

<a name="old-lo"></a>
**LO, lock ownership** (`3a-3-lo.tex:28`). "A memory access instruction need not be instrumented if all accessed
locations are owned by held locks." Interprocedural must-held lockset, intersected per location; global variables.

```c
pthread_mutex_lock(&lock);
g_counter++;                        // before: __tsan_write4(&g_counter);   after: nothing
pthread_mutex_unlock(&lock);        // (every other access to g_counter also holds `lock`)
```

<a name="old-ea"></a>
**EA, escape analysis** (`3a-4-ea.tex:1-29`). "Accesses to memory objects that are known to be confined to a single
thread cannot be involved in any races." An object escapes when its pointer is stored to a global, stored into an
escaped object, passed to a callee where the parameter escapes, or returned. Stack and heap objects.

```c
void handle(void) {
    req_t req;                      // never stored to a global, a shared object or another thread
    req.id = next_id();             // before: __tsan_write4(&req.id);      after: nothing
}
```

<a name="old-de"></a>
**DE, dominance elimination, with peeling** (`3b-redundancy.tex:56-66`). An access `a` is redundant if an access
`a_before` dominates it, both access the same location, `a_before` is a write if `a` is, and all paths between them
"are free of release-like instructions and external calls". Peeling (`3bb-transformations.tex:3-7`) splits off the
first iteration of a loop so that its check dominates the rest.

```c
x = p->len;                         // __tsan_read4(&p->len)
y = p->len + 1;                     // before: __tsan_read4(&p->len);  after: nothing
```

<a name="old-dynstc"></a>
**DynSTC** (`3bb-transformations.tex:11-78`), introduced as STC's "dynamic counterpart": "At access instructions, we
then branch on whether `__thread_cnt > 1`". The only run-time test in the submitted paper.

```c
if (__tsan_thread_count > 1) __tsan_write4(&g);   // before: __tsan_write4(&g) unconditionally
g = 5;
```

<a name="new"></a>
## 3. The new optimizations, each under the analysis it extends

Each entry: what it does, the example, the result, why it belongs under that analysis, and the objection to expect.
Results are changes of speed from adding the optimization, as in `optimization-results.md` section 3, unless a
speedup over stock (×) is given.

### Under STC and DynSTC

<a name="dynstc-rt"></a>
#### DynSTC-RT — direct

DynSTC's test moved from the compiled code into the runtime: while one thread is alive, the runtime records nothing.
The binary is unchanged; it is an option of the runtime.

- Result: FFmpeg +12.0 % (Intel; the paper's compiled guard: +10.1 %); the other applications within about ±2 %.
- Why here: it is the same counter and the same condition as DynSTC; only the place of the branch differs.
- Objection: it is a runtime change, not instrumentation. Answer: the submitted DynSTC already changes the runtime
  (it maintains the counter there).

<a name="n1-st"></a>
#### N1-ST — direct

DynSTC's guard placed around the inline test of [N1](#n1).

```c
// N1:     if (no_matching_record(p)) __tsan_read4_miss(p);
// N1-ST:  if (multithreaded && no_matching_record(p)) __tsan_read4_miss(p);
```

- Result: FFmpeg +16.9 % (Intel) over N1 with DynSTC-RT; nothing elsewhere.
- Why here: literally DynSTC applied to the check in its inlined form. One sentence in the DynSTC text.

<a name="quiet"></a>
#### Quiet threads — stretched

DynSTC asks "is there only one thread?". Quiet threads asks "is only one thread active?". Redis's main thread does
nearly all the work while its I/O and background threads are parked. The compiler derives from the whole program
where the other threads park; while every one of them has been parked since a lock release that the main thread
acquired, the main thread skips its checks.

```c
// main thread, per command:
if (all_other_threads_quiet()) {    // guard
    reply->len += n;                // unchecked
    memcpy(reply->buf, s, n);       // unchecked
} else { /* the same code with the checks */ }
```

- Result: Redis +31.9 %; it is most of Redis's 1.54×.
- Why here: the paper motivates DynSTC with a related case, thread pools whose threads have been joined
  (`3bb-transformations.tex:19-21`). Quiet threads goes one step further, from "the other threads are gone" to "the
  other threads are parked", and the parking points come from a static whole-program analysis, so it is a static +
  dynamic pair like STC + DynSTC.
- Objection: the threads are alive, so DynSTC's theorem does not cover it; the argument needs the happens-before
  edge through the lock the main thread acquires, and premises that were checked for Redis only. It needs its own
  statement. The summary step adds compile time (it was 7.9 s and is now 4.6 s of Redis's 14.4 s build; most of
  Redis's +168 % is [N1](#n1)).

### Under SWMR

<a name="swmr-roots"></a>
#### SWMR-ROOTS — direct

SWMR with "before the threads" counted per thread entry function instead of from `main`: reads of a global need no
checks if every write to it precedes the creation of each thread that can reach it, even if other threads exist.

```c
settings.port = 11211;              // every write is before the create
pthread_create(&t, 0, worker, 0);   // worker and what it calls are the only readers
// in worker:  use(settings.port);  // before: __tsan_read4(&settings.port);  after: nothing
```

- Result: memcached +0.9 %; inert on Chromium (needs the whole program).
- Why here: the same proposition with a finer notion of ST context. One paragraph in the SWMR subsection.
- Objection: none expected; the gain is small, so it should take little space.

### Under LO

<a name="lo-obj"></a>
#### LO-OBJ (LO-OBJ-G; variant LO-MARK) — stretched

The paper's LO proves, for a global variable, that one lock is held at every access. LO-OBJ is LO for heap objects,
where the lock is a field of the object's owner and differs per object: a spec says which lock protects which
object, and an access is unchecked while the thread holds that lock, tested at run time. The spec is generated from
the program's own assertions such as `assert(sqlite3_mutex_held(pBt->mutex))` (or written by hand). LO-MARK is a
variant that needs no assertions; it is measured but not adopted.

```c
sqlite3_mutex_enter(pBt->mutex);
pBt->nPage++;                       // before: __tsan_write4(&pBt->nPage)
                                    // after:  if (!held(pBt->mutex)) __tsan_write4(&pBt->nPage);
sqlite3_mutex_leave(pBt->mutex);
```

- Result: SQLite 1.19× with the generated spec (1.25× with a hand-written one). It is SQLite's whole figure; the
  paper's static LO finds almost nothing there.
- Why here: the same property, "the location is owned by a held lock" (`3a-3-lo.tex:28`). Static LO establishes
  ownership by intersecting locksets; LO-OBJ takes the ownership relation from the program's assertions and tests
  "held" at run time. It is to LO what DynSTC is to STC.
- Objection, the main one: LO *proves* ownership, LO-OBJ *assumes* it. The run-time test shows that this thread
  holds the lock, not that every other access to the object does. If the program breaks its own locking discipline
  where it has no assertion, the race can be lost; this is a stated premise in `soundness-fixes.md` (ownership per
  type), not a theorem. Second: the paper calls the approach "fully automatic" (`1-introduction.tex:149`), true in
  that nobody writes annotations, but programs without such assertions get nothing from it.

### Under EA

<a name="ea-contents"></a>
#### EA-CONTENTS — direct

A refinement of the escape rules: reading a pointer out of a private container and publishing that pointer no longer
makes the container itself escape.

```c
item *slots[8];                     // private array
publish(slots[i]);                  // the item escapes; the array does not
slots[j] = 0;                       // before: __tsan_write8(&slots[j]);  after: nothing
```

- Result: memcached +1.4 %.
- Why here: it sharpens the "transitive escape" rule. One or two sentences in the EA subsection.

<a name="memintr"></a>
#### MEMINTR — by idea, partly stretched

A `memcpy` whose source is a private object or a constant checks only its destination.

```c
char buf[64];                       // private to this thread
memcpy(shared->data, buf, 64);      // before: both ranges checked;  after: only shared->data
```

- Result: within ±2.5 % alone (not beyond the noise); part of Chromium's CR1 together with [FE-INL](#fe-inl) (+9.8 % on Intel and +7.4 % on AMD
  for the pair; MEMINTR's own share there was not separated).
- Why here: for a private source it is EA's fact applied to range checks, which the submitted pass did not touch.
- Objection: for a constant source it is not EA (stock TSan already skips reads of constant globals; this extends
  that to intrinsics). Honest wording: "the analyses' results are also applied to memory intrinsics".

<a name="evconf"></a>
#### EVCONF-CHECKED (annotated form: EVCONF) — stretched

EA proves that an object never leaves its thread. In an event-driven server most hot objects do leave the thread that
allocated them once (the listener hands a connection to a worker) and are then used by one thread. EVCONF-CHECKED
elides checks on such memory: the analysis proposes the candidates, and run-time owner checks plus a marker in the
shadow make each elision sound. A failed owner check voids the run instead of losing a race silently. EVCONF is the
earlier form with seven annotated fields.

```c
char *p = c->rcurr;                 // c->rcurr itself: checked as before
n = p[0];                           // before: __tsan_read1(p)
                                    // after:  if (!owned_by_this_thread(p)) __tsan_read1(p);
```

- Result: memcached 1.80× over stock, no annotations (1.81× with the seven annotations).
- Why here: the same property as EA, "confined to a single thread" (`3a-4-ea.tex:2`), established dynamically where
  the static escape rules must give up. Section 4 of the results shows why it is needed: of the executed checks on
  memory that is in fact used by one thread or consistently ordered, the share on heap memory reachable from shared
  structures is 79 % on SQLite, 42 % on Redis, 34 % on memcached (7.5 % on FFmpeg; MySQL not measured).
- Objection: these objects do escape in EA's sense; this is ownership transfer, a different mechanism. It needs its
  own soundness statement, and premises of the EVCONF line are among those still undecided
  (`soundness-fixes.md` §6b).

### Under DE

<a name="verified-de"></a>
#### Verified DE — direct

The same set of covered accesses as the paper's DE, but the covered check is not deleted. It becomes a short inline
test that the record of the dominating check is still in the shadow cell, and calls the runtime only if not.

```c
x = p->len;                         // __tsan_read4(&p->len)
y = p->len + 1;                     // before: __tsan_read4(&p->len)
                                    // after:  if (record_of_the_check_above_is_gone(&p->len)) __tsan_read4(&p->len);
```

- Result: speed within about 1 % of deleting (Redis: deleting reads 3.8 % slower in one comparison). DE in either
  form carries none of the adopted figures: Chromium's CR1 has no DE, and SQLite runs at the same speed without it.
- Why here: it is DE made exact for a detector with bounded shadow memory. The submitted DE section argues on an
  idealized detector; TSan keeps four records per cell, so the dominating record can be displaced and deletion can
  lose a race stock reports. This is the content of item 5 of the task list (DE and bounded shadow memory).
- Consequences the team must see: (1) the test is the same one as [N1](#n1)'s, so verified DE is "N1 placed where
  DE proves a hit"; in a build with N1 at every check, DE adds nothing. (2) DE then removes no check, so the paper's
  static reduction figures for DE (share of instrumentation removed) no longer apply to the camera-ready builds.
  (3) Peeling only feeds DE and costs compile time (SQLite +25 % with it, +16 % without); it reads −1.9 % / +1.1 %
  on Chromium (Intel / AMD). Whether DE and peeling stay in the evaluated configurations is an open decision.
- Objection: none on the mapping; the text must admit that the submitted form could lose races through displacement.

<a name="vwide"></a>
#### VWIDE-loops — by idea

Verified DE extended to checks in loops that no earlier check dominates, where an earlier check of the same address
is likely by a static estimate. The check becomes the inline test of [N1](#n1).

```c
for (i = 0; i < n; i++)
    if (v[i] > 0)
        sum += a->total;            // conditional: no iteration's check dominates the next
                                    // before: __tsan_read4(&a->total) each time
                                    // after:  inline test, runtime call only on a miss
```

- Result: with FE-INL, Redis +7.2 % (Intel) and MySQL +4.6 % (FE-INL alone: +4.8 % and +3.6 %). Not in an adopted
  figure.
- Why here: it removes DE's dominance condition and pays for that with a run-time test. It can be one sentence
  between verified DE and N1, or left out.

<a name="n1"></a>
#### N1, inline hit test — by idea (the boldest claim, and the most important one)

Most checks find that the same thread already recorded the same access in the shadow cell, and return. N1 does that
comparison in place and calls the runtime only on a miss. Placement variants (in small functions: S56; by static
frequency: s1, s4; in small loops: K4) put the test at fewer checks to save code and compile time.

```c
x = p->field;
// before:  __tsan_read4(&p->field);
// after:   if (none of the cell's four records == this access by this thread, in this epoch)
//              __tsan_read4_miss(&p->field);
```

- Result: Chromium on Intel +16.5 % (first sample) and +13.4 % (hold-out) over CR1; on AMD +1.2 % and −3.8 %.
  FFmpeg +5.0 % (Intel). Over the plain analyses the other applications are within ±2.5 %, but on top of the
  ownership configurations it loses: −5 % on memcached's, −18.5 % on SQLite's.
- Cost: 133–140 bytes per check. Chromium: code 3.0× stock's, peak memory of the largest process 1.77×. Compile
  time +97 % on FFmpeg, +10 % on Chromium, and most of Redis's +168 %. Implemented for x86-64 Linux and aligned
  accesses of 1 to 8 bytes. Unexplained and open: two server deaths in 104 MySQL runs with it (none in 80 later
  runs), and a lower report rate of one Chromium race.
- Why here: the test is DE's redundancy condition evaluated at run time. DE's four conditions say: an earlier access,
  the same location, at least as strong (a write if this is a write), no release-like instruction in between. A
  matching shadow record says the same: this thread accessed this location with this kind of access, and its epoch
  has not advanced, which TSan does at release-like operations. DE proves it for all paths and needs dominance and
  must-alias; N1 observes it on the path taken and needs neither. So **N1 is to DE what DynSTC is to STC**, and the
  placement variants are the static half: an analysis decides where the dynamic test pays.
- It also gives the paper a better answer on ReX (`4-experiments.tex:177`): a dynamic redundancy filter with a
  near-zero per-access cost, because it reuses the shadow state TSan already keeps.
- Objection: stock TSan's runtime already performs this comparison as its first step, so a reviewer can call N1
  "inlining the fast path", an engineering change. It also removes no check, it makes the common one cheaper. The
  honest wording is "dynamic redundancy test, inlined", not "elimination". If the team finds this too thin, N1 goes
  with FE-INL under the fallback of [section 4](#nomap).

<a name="nomap"></a>
## 4. What does not map

<a name="fe-inl"></a>
#### FE-INL, inlined function entry and exit

TSan keeps a shadow call stack for its reports and calls the runtime at every function entry and exit. FE-INL does
the push and the pop in place.

```c
int get(obj *o) {
    __tsan_func_entry(pc);          // after: *thr->shadow_stack_pos++ = pc;   (a few instructions)
    ...
    __tsan_func_exit();             // after: thr->shadow_stack_pos--;
}
```

- Result: MySQL +3.6 % (its 1.13× configuration), Redis +4.8 % (Intel), Chromium +9.8 % on Intel and +7.4 % on AMD
  (with MEMINTR, = CR1).
- Why it does not map: function entry and exit are not access checks, and none of the paper's analyses reasons about
  them. It cannot be left out: MySQL's, Redis's and Chromium's figures depend on it.
- Fallback (item 4 of the task list): one paragraph in the transformations section (`sec:trans`, "Code
  Transformations to (Dynamically) Reduce Instrumentation Overhead"), or in the pipeline section, whose remark
  already says "The pipeline can be extended with further analyses" (`3c-pipeline.tex:25`). It needs no
  soundness argument (the recorded stack is the same) and about ten lines of text.

Not proposed for the paper (measured, not in any adopted figure): FE-SINK, N1-L and the placement rule k2, LO-MARK,
the annotated EVCONF variants (RANGES, ARGS, INTERCEPT) beyond one sentence, Chromium's library-facts table (reads
the same with and without), and the families of `optimization-results.md` section 4.

<a name="structure"></a>
## 5. What the paper would look like

| paper section | change | new text (rough) |
|---|---|---|
| Introduction, contributions | "the five static analyses described and the dynamic optimization" becomes the five static analyses and, for four of them, a run-time counterpart where the static proof gives out | rewrite of ~15 lines |
| STC, LO | unchanged | — |
| SWMR | one paragraph: thread roots | ~6 lines |
| EA | two sentences: contents rule; results applied to memory intrinsics | ~5 lines |
| DE (`sec:beyond`) | verified form and bounded shadow memory (item 5) | ~15 lines |
| `sec:trans`, now "Run-time counterparts of the analyses" | DynSTC (existing text, plus the runtime form, N1-ST, quiet threads); dynamic lock ownership (LO-OBJ); dynamic confinement (EVCONF-CHECKED); dynamic redundancy test (N1, placement); inlined entry/exit (FE-INL) | ~1.3 columns new; the largest item |
| Pipeline | figure and order updated; summary step for the whole-program parts | ~8 lines, figure edit |
| Evaluation | configurations renamed after the pairs; per-app adopted configuration; compile time | separate items 2 and 7 |

Total new text in section 3: about 1.5 to 2 columns, against the page limit (item 6).

<a name="open"></a>
## 6. Open decisions

1. **N1 under DE, or under the fallback with FE-INL?** The argument for DE is in [its entry](#n1); the risk is the
   "it is only inlining" objection. Chromium's and FFmpeg's figures depend on N1 either way.
2. **How to state the result of the analyses alone.** With the soundness fixes, the four analyses and DE read about
   1.0× on SQLite, memcached and Chromium; the adopted figures come from the extensions above. Proposed wording:
   the paper reports "the analyses with their run-time counterparts" as one system and gives the static-only
   configuration as an ablation row. The figures are not softened.
3. **"Fully automatic" and generality.** [LO-OBJ](#lo-obj) uses the program's own assertions; [quiet threads](#quiet)
   and [EVCONF-CHECKED](#evconf) need a whole-program summary step. Each of the three was built for and measured
   on one application. The claim holds as "no annotations written for the tool", and should be worded so.
4. **DynSTC-RT is a runtime option.** 1.085× of FFmpeg's 1.29× is this option alone. The submitted DynSTC already
   touches the runtime, so it fits, but the text should say where the branch lives.
5. **Soundness statements owed.** Quiet threads, LO-OBJ and EVCONF-CHECKED each need a stated premise and argument
   (the paper has theorems for the others). LO-OBJ assumes ownership instead of proving it. Several premises are
   still pending a decision (`soundness-fixes.md` §6b lists them; some belong to the EVCONF line and one to quiet
   threads). A failed owner check voiding the run is a behaviour the submitted paper does not have.
6. **Two hosts.** N1 pays on Intel and not on AMD for Chromium. The other applications' adopted figures are from
   AMD, while several of the single-optimization figures quoted here for FFmpeg and Redis are from Intel. The paper
   must name the host per figure.
7. **DE and peeling in the evaluated configurations.** See [verified DE](#verified-de): keep them as a static-only
   ablation, or as part of the adopted configurations where they are on.
8. **Space.** If 1.5 to 2 columns do not fit, the candidates to shorten are VWIDE-loops (drop), MEMINTR and
   EA-CONTENTS (one sentence each), SWMR-ROOTS (two sentences).
