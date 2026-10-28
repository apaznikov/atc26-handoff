# Messages to the contributors (drafts of 26 Oct; not needed)

28 Oct: the imported branches stay as they are; Alexey tells the contributors himself that everything is ready.

To be sent once Alexey decides the target branches (reset `*/main` to base, or new `<folder>/dev`). Replace the
target branch names below if `/dev` is chosen. Fork names: `<fork>` = the contributor's GitHub account.

## Ownership Meta-Shadow Memory

> Hi,
>
> Your project is now part of our TSan work in **focs-lab/llvm-project-tsan** (public), under the branch folder `meta-shadow/`: `meta-shadow/base` is your base commit (73e729f7b2f0); `meta-shadow/parallel-note`, `meta-shadow/shadow-resident-filter` and `meta-shadow/workload-profiler` are the targets for your three lines of work.
>
> Please open three pull requests, one per branch, from your fork into ours (same llvm-project fork network, so no access is needed):
> `gh pr create --repo focs-lab/llvm-project-tsan --base meta-shadow/parallel-note --head <fork>:feat/parallel-note --title "..." --body-file pr.md`
> (likewise for feat/shadow-resident-filter and feat/workload-profiler).
>
> In each PR description: what the change does and why; how to build and run it and which tests cover it (negative tests too); the results with their methodology (host, repetitions, spread); known limitations and any case where it could miss a race stock TSan reports. Link the documentation in your docs repository; docs, profiler and scripts can stay there.
>
> From now on, keep working in your fork and send changes as PRs into these branches: one logical change per PR, with tests, commit messages in English saying why. Do not rebase or force-push a branch while its PR is open. Contributions are under LLVM's license (Apache 2.0 with LLVM exception).

## L0 filter

> Hi,
>
> Your project is now part of our TSan work in **focs-lab/llvm-project-tsan** (public), under `l0-filter/`: `l0-filter/base` is your base commit (4d87e55d8a41), `l0-filter/main` the target branch.
>
> Please open a pull request from your fork (same fork network, no access needed):
> `gh pr create --repo focs-lab/llvm-project-tsan --base l0-filter/main --head <fork>:L0Filter --title "[TSan] L0 filter: compiler-inlined thread-local memoization" --body-file pr.md`
>
> In the description: the design (what the per-thread cache holds and when an entry becomes invalid: synchronization, epochs, slot resets, free/malloc); why a skipped check cannot lose a race stock TSan reports; how to build and test it (pass tests and runtime tests; please add negative tests that catch a stale entry); results with methodology; known limitations.
>
> Keep working in your fork and send changes as PRs into `l0-filter/main`: one logical change per PR, with tests, English commit messages. Do not rebase or force-push while a PR is open. Contributions are under LLVM's license.
>
> About running benchmarks on the focs server: it is fully booked for timed measurements this week; we will come back to you with an arrangement.

## ABA detection

> Hi,
>
> Your project is now part of our TSan work in **focs-lab/llvm-project-tsan** (public), under `aba/`: `aba/base` is your base commit (8947e494f92e), `aba/main` the target branch.
>
> Please open a pull request from your fork (same fork network, no access needed):
> `gh pr create --repo focs-lab/llvm-project-tsan --base aba/main --head <fork>:tsan-aba-pool --title "[TSan] Detect heap-reuse ABA on atomic CAS" --body-file pr.md`
>
> The description can follow your notes: what is detected, the allocation-epoch design (8-bit heap epochs, the pool annotation API), the build and test commands (`LIT_FILTER=aba_ ... check-tsan`, the Treiber smoke test), the known limitations (`aba_lim_*`, a possible missed report on an epoch collision) and why no false positives are expected, and the overhead on programs without ABA if measured.
>
> Keep working in your fork and send changes as PRs into `aba/main`: one logical change per PR, with tests, English commit messages. Do not rebase or force-push while a PR is open. Contributions are under LLVM's license.

## Technical next steps from the audits (to add if Alexey wants)

- Meta-shadow: lift the leaf-only rule (fast state per access), add `_miss` entries and a start-up layout check (a portable N1); negative tests with controls; fix the two parallel-note holes and keep it as a negative result; build the fifth-slot design.
- L0 filter: make it build on Linux and default off; one negative test per hole with a control; tag entries with TSan's fast state, key on exact cell/bytes/type, add a global generation at reset and free; then compare against N1.
