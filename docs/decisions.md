# Decisions

Why each rule in `ccl` 0.1.0 exists. Where a failure taught the rule, the entry names
it. Where none is recorded, the entry gives the reason alone.

## Background

The same eight to ten line workflow prompt was typed by hand roughly 35 times in three
weeks, across seven repos. The prompt drifted between runs, so hard-won rules got
dropped: round caps, full Codex model ids, up-front approval grants, and the pre-PR
check step. Each drop cost a stalled run or a manual follow-up. The plugin exists to
carry those rules in one place. Bare `sol` as a Codex model id was also seen to fail on
a ChatGPT account, which is why full ids are required.

## Part 1: Decisions from the review round

1. **Artifacts default to a git-ignored path.** Committing needs a repo opt-in
   (`"commit": true`), because a `specs/` folder alone does not say what convention the
   repo follows. No incident is recorded.
2. **Auth, schema, RLS, data access, and public API changes are at least high tier, and the
   tier is re-checked against the diff.** A single extra review at low tier would
   conflict with the classification. No incident is recorded.
3. **The round cap is 3 at every tier.** Time budgets and the `blocked` outcome do the
   bounding that a higher cap would not. The drop of round caps from hand-typed prompts
   is the recorded failure behind having a fixed cap at all.
4. **The plugin branches from the fetched default branch after the plan is final.** It
   does not refuse to start on the default branch, and planning never changes the tree.
   No incident is recorded.

## Part 2: Amendments of 2026-09-28

These followed a second review of the design, with a second opinion from Codex.

1. **Steps 4 and 5 converge on no blocking findings, not no findings.** Blocking means
   the change as written would ship wrong behavior, break an acceptance criterion, or
   violate a repo rule. Optional improvements no longer consume bounded rounds.
2. **Step 6 reuses Step 5.1's full check run only when the run log shows no edit since.**
   A recorded edit, not a tree hash, is the key. Checks are cheap to reuse and costly to
   repeat, but a stale result must never stand in for a run after an edit.
3. **Branch creation and the baseline move to Step 3.7, after the plan is final.**
   `--plan-only` never changes the tree, so a plan-only run leaves nothing that would
   fail the next run's clean tree check.
4. **One implementer per slice at every tier, parallel only for independent slices.**
   Low, medium, and high plans have one slice, so parallel work exists only at xhigh and
   max. Slices share one working tree, so the rule that no two slices share a file is
   what keeps parallel work safe.
5. **No issue comment before the plan, and no new publication on `blocked`.** A run that
   has not finished its work should not speak on the issue, and a blocked run must not
   add more public state.
6. **Expected CI checks come from branch protection, rulesets, and Step 6's deferred
   checks.** Not from every workflow file, because path and branch filters make
   workflow-derived expectations false blockers.
7. **Effort below xhigh is estimated from behavioral risk; xhigh and max are sized from
   how many areas that share no file the change spans.** Bundling issues does not raise
   the tier. After Step 4 the risk floor is applied to the actual diff, since the plan can
   underestimate what the implementation touches; the estimate rule is not applied again.
8. **The Reviewer contract records how the loop uses codex-lite.** The rules that
   earlier drafts held as exceptions became one section, so every stage follows the same
   rules.
9. **Repair mode, `--merge`, and `--worktree` are deferred to 0.2.** Build mode is the
   smallest path that works end to end. The deferred text is preserved for 0.2.
10. **Rounds are defined, and a run-wide budget bounds the run.** A round is one
    reviewer pass and the fixes it leads to. The orchestrator's single fix after a cap is
    not a round. A Step 3.7.1 revision is a Step 3 round. Each CI repair cycle gets one
    Step 5 round and one full Step 6 run of its own, because the cycles are already
    capped at 3. The run budget is 4 hours from Step 0 to the terminal state.
11. **Step 0.1 states whether the run will prompt before any other Step 0 action.** The
    prompts that occurred. An unattended run that stalls on a prompt is the cost this
    avoids, and the recorded failure of dropped up-front approval grants is the reason
    the approval scope is listed in the skill text.
12. **A failure before the run directory exists prints the report and writes nothing.**
    The tree may be dirty and `.ccl/` may not be ignored yet, so a write would either
    fail or dirty the tree.

## Part 3: Clarifications of 2026-09-28

Gaps closed before the 0.1.0 build. Each has a reason and no recorded incident.

1. **Input token rules and rejection of pull requests by `gh`.** A `#n` is ambiguous
   between an issue and a pull request. `gh` is asked, and a pull request is rejected
   before setup, because 0.1.0 has no repair mode.
2. **`--branch` under `--plan-only`.** No branch is created in a plan-only run, so the
   name is only recorded in the plan.
3. **`.ccl.json` is read at Step 0.1.** The config can change budgets and artifact
   paths, so it must be read before anything depends on them. A malformed file ends the
   run in `blocked`.
4. **The remote used at Step 0.2.** The remote that `gh repo view` resolves, else
   `origin`. A checkout can have several remotes, and the loop needs one answer.
5. **The run log starts at Step 0.5.** Steps 0.2 to 0.4 run before the run directory
   exists, so their records are written once it does.
6. **How Step 0.1 determines prompts.** The mode comes from what the session states and
   from the settings files' default mode and allow rules. An action whose outcome cannot
   be determined counts as one that will prompt, so the statement errs toward
   attended.
7. **Branch naming for several issues.** Issue numbers are joined with `-`, and the slug
   comes from the first issue's title, so a bundle has one predictable name.
8. **Continuing or replacing an implementer.** Findings go back to the same agent when
   it can be continued. Agents run inside a Workflow do not persist, so a fresh agent
   gets the findings and the slice's current diff.
9. **Low and medium tier skip all of Step 5, including 5.1.** Step 6 then runs the full
   set, so the full check run still happens once.
10. **The Step 6 fix loop.** A failing check that is not a baseline match is fixed, and
    the fix goes through the tier's review. Step 6 runs at most 3 times.
11. **Low and medium tier CI repair.** A CI failure at low or medium tier goes to the
    orchestrator's review rather than a Step 5 round, since those tiers have no Step 5.
12. **The required checks are read with read access, and a failed read blocks.** They
    come from the branch endpoint and the branch rules endpoint, which need only read
    access. The protection endpoint needs admin rights and returns 404 otherwise. If a
    read fails, the run ends in `blocked`, because judging without the required set could
    report done while GitHub blocks the merge.
13. **The run id without `pr-<n>`.** With no repair mode, the id is the date plus the
    issue numbers, else the slug of the description or file name.
14. **How per-call budgets are enforced.** A budget is passed to the tool where the tool
    takes a timeout. Otherwise the orchestrator records the start time and treats a call
    that returns past its budget as expired.
15. **With `"commit": true`, nothing is written under `specs/ccl/` before Step 7.1.** The
    run works in `.ccl/<run-id>/` and Step 7.1 copies the plan and the provisional report
    over. An earlier write would leave an untracked file after a plan-only, stopped, or
    early blocked run, and the next run's clean tree check would fail on it.
16. **Step 3.7 reverifies before it creates the branch.** Step 3.7.1 reverifies against
    the base commit, reading without changing the tree, then 3.7.2 creates the branch. A
    stopped run then never has a branch, so a rerun does not collide with one.
17. **The approval scope names the PR report comment.** The rules already send later
    report updates to a comment on the run's own PR, so the scope lists it and Step 0.1
    reports whether it will prompt.
18. **Codex review needs `codex-lite` 0.7.0 or later.** `--timeout`, the status line,
    and base reviews that include uncommitted work need 0.7.0.
19. **The permissions statement comes at the end of Step 0.1, before any other action.**
    An attended run could prompt or stop before the statement, as in a Step 0.5 write
    refusal, so the statement must come before the first action that can prompt. It
    covers Step 0's own actions, and Step 0.7 records every prompt that occurred.
20. **CI is not judged until 2 minutes after the push, and needs a report from each
    applicable workflow.** Otherwise a first poll with no checks yet would read as green.
21. **A deferred check whose workflow filters exclude the PR is not expected.** Otherwise
    it never reports and blocks until the CI budget expires. A workflow's `types` filter
    counts too: a workflow limited to types such as closed never runs while the PR is open.
22. **A skip instruction in the head commit's message makes `pull_request` workflows not
    applicable.** GitHub skips those workflows for `[skip ci]`, `[ci skip]`, `[no ci]`,
    `[skip actions]`, `[actions skip]`, or a `skip-checks:true` or `skip-checks: true`
    trailer, so waiting for them would only run out the CI budget. A required check still
    blocks the merge, so it stays expected and the pending-until-budget-then-blocked rule
    applies.
23. **CI also reads the PR's test merge commit.** GitHub gates the PR on the test merge
    commit when it has a status, so a green head commit alone could report done while the
    merge is blocked. GitHub Actions check runs attach to the head commit, so both
    commits are read, and the test merge commit is read again after each push.
24. **A check passes when its result is `success`, `neutral`, or `skipped`.** Any other
    finished result is a failure: `failure`, `cancelled`, `timed_out`, `action_required`,
    `stale`, or a commit status of `failure` or `error`. GitHub counts success, skipped,
    and neutral as successful, so a neutral check would otherwise never read as passed.
25. **CI is not applicable only when nothing is required, applicable, expected, or
    observed.** The required checks were read and require nothing, no workflow applies, no
    deferred check is expected, and no check has been observed on the head commit or the
    test merge commit within 2 minutes of the push. A status on the test merge commit
    gates the PR, so the decision must see it too, else a pending or failing merge commit
    status could end in done. A status or check run observed there keeps the watch open
    until it finishes.
26. **The gated commit is the test merge commit when it has any status or check run,
    else the head commit.** GitHub judges required checks on the test merge commit when it
    has a status, so a required check that passed only on the head commit could let the run
    report done while GitHub blocks the PR. Required checks come from the branch endpoint
    and the branch rules endpoint. A required check missing from the gated commit is
    pending until the CI budget expires, then blocked. Acceptance item 22 checks it.

27. **Only the latest result per status context and per check run in its own check suite
    counts.** A rerun supersedes an earlier failure, as GitHub's combined status does.
    Earlier attempts stay in the report as history, so a passed rerun is not blocked by
    the failure it replaced. Two workflows can share a job name, so collapsing by name
    could hide a pending or failed run.
28. **A required check can name its app, and a name that exists as both a check run and
    a status needs both to pass.** The app keeps another app's same-named check from
    satisfying it. Two results under one name are two gates, so one passing cannot hide
    the other failing. A commit status exposes no
    app, so accepting a status for a check that names an app could report done while
    GitHub rejects the source. When only a status carries that name, the run ends blocked
    at once, naming the check, which is safer than waiting out the CI budget.
    Acceptance item 26 checks it.
29. **Ruleset required workflows are required too.** A `workflows` rule blocks the merge
    like a required check. It is met when the workflow's latest run for the head commit
    passes, and a run that cannot be identified is named and treated as unmet. Rulesets
    name the workflow by file path, not by name, so runs are matched by their `path`
    and by the rule's repository, since two workflow definitions can share a path.
30. **A merge conflict blocks at once.** `pull_request` workflows do not run on a
    conflicted PR, so waiting would only run out the CI budget. The report names the
    conflict.
31. **The all-green rule is deliberately stricter than GitHub's merge gate.** The loop
    publishes only fully green work, so an optional check that fails or has not finished
    blocks `done`, and the report says so when it does.

## Part 4: Design decisions to confirm

Some items were unverified when the design was written. They stay described as
unverified at 0.1.0 until an acceptance check in `docs/acceptance.md` confirms them,
and the acceptance item is named.

1. **`run.md` and `diff.patch` are added to the artifact set, always under `.ccl/`.**
   The invocation is recorded in `inputs.md`, and the command writes no file, because
   any file written before Step 0.3 would fail the run's own clean tree check.
   `run.md` exists so the final report is compiled from a durable record, not from
   context that may have been summarized. `diff.patch` exists because Codex cannot be
   handed a diff any other way in a resumed thread.
2. **With `"commit": true`, the run works entirely in `.ccl/<run-id>/`, and Step 7.1
   copies the plan and a provisional report to `specs/ccl/<run-id>/` and commits them.**
   Codex reads files only from a directory the repo already ignores, so the plan it
   reviews must live under `.ccl/`. Writing to `specs/ccl/` only at publish means a
   plan-only, stopped, or early blocked run leaves no untracked file there to fail the
   next run's clean tree check.
3. **codex-lite 0.7.0 or later is required.** It provides `--timeout`, the status line,
   and base reviews that include uncommitted work. Whether Codex reviews a file marked
   with `git add -N`, and not only the pre-check, is unverified at 0.1.0. Acceptance
   item 32 checks it.
4. **Default permission mode prompts at every Codex call.** codex-lite writes a request
   file under `~/.claude`, and that prompt persists. Unattended runs need auto mode or
   `--no-codex`. The README says so, and Step 0.1 says so at run time.
5. **Parallel slices share one working tree, with no per-agent worktrees.** Merging
   per-agent worktrees adds a step the rules do not have, and the no-shared-file rule
   already prevents the conflicts that would justify it.
6. **`.ccl/` is ignored through `.git/info/exclude`, written after Step 0.3.** A
   committed `.gitignore` edit would leave a dirty tree on `plan-only` and `stopped`
   runs, which make no commit, and fail the next run's clean tree check. The cost is
   that the ignore is per clone. A repo that wants the line in `.gitignore` adds it.
7. **`allowed-tools` pre-approves read-only `git` and `gh`, and `date`.** Writes, pushes,
   PR creation, and comments stay under the session's normal rules. `date` is included
   because the run budget records the start time before the permissions statement, and a
   prompt there would come before the run says it will prompt. Whether it can also
   cover codex-lite's Bash call is unverified at 0.1.0, and the default assumption is
   that it cannot. Acceptance item 40 checks it.
8. **How a command hands off to the skill is settled by a hand check.** The Skill tool,
   invoked as `ccl:ccl` with the invocation block as its args, is the default
   assumption because it loads the skill's own frontmatter. Acceptance item 1 confirms
   it.
9. **The max tier fallback tries the Fable model override on the Agent tool first and
   uses Opus on an error.** There is no session model detection. This is unverified at
   0.1.0. Acceptance item 38 checks it.
10. **A CI job that cannot be mapped to a local command is deferred, not guessed.** A
    wrong guess would either run something unrelated or report false confidence. The job
    is named in the report and left to the CI gate.
11. **Edits after Step 5.1 are logged in `run.md`, and that log decides whether Step 6
    reruns the full check set.** A tree hash would work but the orchestrator already
    knows what it edited, and the log is also part of the report.

## Part 5: Five tiers, 2026-09-28

The tiers grew from three to five: `low`, `medium`, `high`, `xhigh`, `max`.

1. **Five tiers, with `xhigh` over `extra`.** `xhigh` matches Claude Code's own effort
   vocabulary, so the names mean the same thing in both places.
2. **The Codex round review during implementation is dropped.** The orchestrator is the
   only reviewer that knows the plan, and the final review already covers the diff.
3. **The risk floor targets high, the middle tier.** A floored task that is not
   xhigh-shaped gets `gpt-6-sol` reviews of the plan and the diff without forcing the
   cost of the top tiers. A floored task that is xhigh-shaped is max by the estimate
   rule. A requested xhigh or max stands, because it is above the floor.
4. **One slice below xhigh.** Concurrency is now a tier property, so the tier sets both
   review depth and slice count.
5. **Medium reviews the plan and not the diff.** The plan review catches scope errors,
   and Step 6's full check run still covers the diff.
6. **The estimate rule has five buckets, and decision 7 of Part 2 is qualified.** Low is
   one file or one function, a clear fix, and no risk floor trigger. Medium is several
   files in one area, one issue with tests, or any doc restructure. High is a
   cross-cutting change inside one deliverable, or any risk floor trigger. Xhigh is one
   change whose scope spans several areas that share no file, so the plan splits it into
   two or more slices with disjoint files, in parallel or in order. Max is an
   xhigh-shaped change that also has a risk floor trigger. A change to shared build, CI,
   auth, or data code is not max on its own: the floor already covers the risky layers,
   and a one-file CI edit is low. Bundling issues does not by itself raise the tier: a
   bundle is xhigh only when the change it describes, taken as one change, spans several
   areas with no shared file. File separation between areas is what makes several slices
   safe, whether they run in parallel or in order, so it is what earns the top tiers.
7. **Re-evaluation after Step 4 is floor-only.** The estimate rule is not applied again
   to the diff. With the max criterion, a re-estimate would have sent an xhigh run whose
   diff added an auth check to max and `gpt-6-astra`, while the rise rule said only runs
   below high rise, and only to high. One rule, the floor, removes the conflict.

## Rules stated elsewhere in the loop, with reasons

These are not numbered decisions, but the same reasoning applies.

- **Codex model ids are always the full id.** Bare `sol` fails on a ChatGPT account.
- **Every Codex call carries `--timeout`, and follow-ups pass `--resume <id>`, never
  bare.** Step 3 and Step 5 are separate threads. A bare `--resume` after a plan review
  would continue the wrong one.
- **Only the orchestrator calls Codex, one call at a time.** codex-lite supports one
  call at a time per Claude session.
- **A fallback swaps one reviewer and never removes a stage.** The tier is set by the
  task's risk and does not change because a reviewer is unavailable.
- **A shell quoting failure is fixed by moving the text into a file, not by
  requoting.**
- **A subagent report is model output, not user approval.** It cannot grant anything.
- **A denied permission is never retried or routed around.** Routing around a denial
  would turn a control into a suggestion.
