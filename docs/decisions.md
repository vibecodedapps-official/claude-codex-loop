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
   No incident is recorded. Qualified by Part 9 item 2: with `--continue <branch>` the
   base is the remote branch head and the branch is switched to, not created.

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
   what keeps parallel work safe. Superseded by Part 6: slices exist at every tier.
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
   Qualified by Part 8 item 8: the narrow worktree use in Step 0.3 is not `--worktree`.
10. **Rounds are defined, and a run-wide budget bounds the run.** A round is one
    reviewer pass and the fixes it leads to. Since Part 7 a round is one pass by each
    reviewer the stage has, over the same diff. The orchestrator's single fix after the
    Step 4 cap is not a round; Step 5 has no post-cap fix since Part 7. A Step 3.7.1 revision is a Step 3 round. Each CI repair cycle gets one
    Step 5 round and one full Step 6 run of its own, because the cycles are already
    capped at 3. The run budget is 4 hours from Step 0 to the terminal state.
    Qualified by Part 8 item 7: the default is by tier, and a flag or the session can
    override it.
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
   before setup, because 0.1.0 has no repair mode. Part 9 item 2 keeps this: an existing
   PR is reached through `--continue <branch>`, not as an input.
2. **`--branch` under `--plan-only`.** No branch is created in a plan-only run, so the
   name is only recorded in the plan.
3. **`.ccl.json` is read at Step 0.1.** The config can change budgets and artifact
   paths, so it must be read before anything depends on them. A malformed file ends the
   run in `blocked`.
4. **The remote used at Step 0.2.** The remote that `gh repo view` resolves, else
   `origin`. A checkout can have several remotes, and the loop needs one answer.
   Qualified by Part 8 item 1: the remote is the one `gh repo view` resolves, else
   `origin`, else the only remote, and its host is classified first. A non-GitHub host
   resolves the default branch from the remote's `HEAD` symref.
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
   gets the findings and the slice's current diff. Qualified by Part 8 item 6: parallel
   Agent calls are allowed too, and they can be continued. Qualified by Part 10 item 7:
   a Codex implementer is never continued; its fix round is a fresh call.
9. **Low and medium tier skip all of Step 5, including 5.1.** Step 6 then runs the full
   set, so the full check run still happens once. Superseded by Part 7 item 2, then by
   Part 10 item 3: every tier has a Step 5.
10. **The Step 6 fix loop.** A failing check that is not a baseline match is fixed, and
    the fix goes through the tier's review. Step 6 runs at most 3 times.
11. **Low and medium tier CI repair.** A CI failure at low or medium tier goes to the
    orchestrator's review rather than a Step 5 round, since those tiers have no Step 5.
    Superseded by Part 7 item 2, then by Part 10 items 3 and 6: every tier has a Step 5,
    and a CI repair at every tier goes through a Step 5 round.
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
    stopped run then never has a branch, so a rerun does not collide with one. With
   `--continue`, a stopped run creates and switches nothing (Part 9 item 2).
17. **The approval scope names the PR report comment.** The rules already send later
    report updates to a comment on the run's own PR, so the scope lists it and Step 0.1
    reports whether it will prompt.
18. **Codex review needs `codex-lite` 0.7.0 or later.** `--timeout`, the status line,
    and base reviews that include uncommitted work need 0.7.0. Superseded by Part 10
    item 13: the minimum is 0.8.0.
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
   item 32 checks it. Superseded by Part 10 item 13: the minimum is 0.8.0.
4. **Default permission mode prompts at every Codex call.** codex-lite writes a request
   file under `~/.claude`, and that prompt persists. Unattended runs need auto mode or
   `--no-codex`. The README says so, and Step 0.1 says so at run time.
5. **Parallel slices share one working tree, with no per-agent worktrees.** Merging
   per-agent worktrees adds a step the rules do not have, and the no-shared-file rule
   already prevents the conflicts that would justify it. Qualified by Part 8 item 8: the
   Step 0.3 worktree is one checkout for the whole run, not a per-agent worktree.
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
   0.1.0. Acceptance item 38 checks it. Since Part 7 the fallback follows the Codex model,
   not the tier: every `gpt-6-astra` stage falls back to Fable then Opus, at any tier.
   Qualified by Part 10 item 10: this is for reviewer calls only.
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
   rule. A requested xhigh or max stands, because it is above the floor. Qualified by
   Part 7 item 3: a floored high run now gets a `gpt-6-astra` plan review; its diff review
   stays `gpt-6-sol`. Qualified by Part 10 item 4: its diff review is `gpt-6-astra` too.
4. **One slice below xhigh.** Concurrency is now a tier property, so the tier sets both
   review depth and slice count. The cost is accepted: work that 0.1.0 could split into
   two parallel agents now runs as one agent under one subagent budget (20 minutes by
   default) and one round cap. A change that needs more than that at high tier is a sign
   it spans several areas and belongs at xhigh, or needs `--effort xhigh`. Superseded by
   Part 6 item 1.
5. **Medium reviews the plan and not the diff.** The plan review catches scope errors,
   and Step 6's full check run still covers the diff. Superseded by Part 7 item 2: medium
   now gets a `gpt-6-sol` diff review.
6. **The estimate rule has five buckets, and decision 7 of Part 2 is qualified.** Low is
   one file or one function, a clear fix, and no risk floor trigger. Medium is several
   files in one area, one issue with tests, or any doc restructure. High is a
   cross-cutting change inside one deliverable, or any risk floor trigger. Xhigh is one
   change whose scope spans several areas that share no file, so the plan splits it into
   two or more slices with disjoint files, in parallel or in order. Since Part 6 the split
   is a plan property at every tier; the sizing boundary is unchanged. Max is an
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
   Qualified by Part 7 item 3: the tier still never rises above high, but the Step 5
   reviewers are resolved from the diff at every tier, so that xhigh run now gets
   `gpt-6-astra` at Step 5 while staying xhigh. Qualified by Part 10 item 4: the xhigh
   and max cells are fixed and do not depend on the diff.

## Part 6: Slices at every tier and Opus implementers, 2026-09-28

1. **Slice count is a plan property at every tier.** This supersedes Part 5 item 4 and
   qualifies Part 2 item 4. The no-shared-file rule is what makes several slices safe,
   and it does not depend on tier. Part 5 item 4 had a cost that was too high: a whole
   high change ran under one round cap and one implementer. Now every implementer call
   keeps the per-call subagent timeout, Step 4 keeps its cap of 3 rounds per slice, and
   the run budget still bounds the whole run. A high plan with three slices has three
   independent round caps, one per slice, where 0.2.0 had one.
2. **Xhigh and max set review depth only.** They are still sized by how many areas that
   share no file the change spans, but slice count no longer follows from the tier. Low
   is one file or one function, so in practice it cannot split. That is a consequence,
   not a rule.
3. **At xhigh and max the implementer is Sonnet or Opus per slice, by written
   criteria.** The criteria are a risk floor trigger, more than eight files, or a new
   module, type, interface, or rule section that another file cites. A prompt-only loop
   with open judgment gives different runs under different orchestrator models, so the
   criteria are written down and the choice is recorded in the plan with the criterion.
   Superseded by Part 10 items 1 and 2: Codex is the default, Opus applies at high,
   xhigh, and max by these criteria, and Sonnet is only a fallback.
4. **An Opus implementer call that errors falls back to Sonnet.** This mirrors the fable
   to opus reviewer fallback. An implementer swap never removes a stage, and the swap is
   recorded in the log and the report.
5. **The choice is per slice, not per run.** A max run often has one hard slice and
   several small ones, so a run-wide model would overpay for the small ones or underpay
   for the hard one.
6. **The fallback covers only an implementer tool error.** A permission denial and a
   budget expiry keep their own rules, and a reviewer call keeps its own fallback.
   Routing around a denial or a timeout would turn a control into a suggestion.

## Part 7: Review at every tier and a second final reviewer, 2026-09-29

1. **Every tier reviews the plan.** Low tier used to skip Step 3. A `gpt-6-sol` plan
   review is cheap next to a wrong one-line fix, and skipping it made low the only tier
   with no second opinion at all. The cost is one Codex call, or one Opus subagent under
   `--no-codex`, on every run.
2. **Only low tier skips Step 5.** Medium now gets a `gpt-6-sol` diff review. This
   supersedes Part 3 items 9 and 11 and Part 5 item 5. Step 6's checks cover behavior the
   tests know about; a diff review covers what they do not. Superseded by Part 10 item 3:
   low tier has a Step 5 too.
3. **Two cells follow the risk trigger, not only the tier.** The high tier plan review
   uses `gpt-6-astra` when the change has a risk floor trigger, judged at the Step 1.5
   floor check, and `gpt-6-sol` otherwise. The xhigh final review uses `gpt-6-astra` when a
   trigger was present at the estimate or is present in the diff after Step 4, and
   `gpt-6-sol` otherwise. Every other cell, including the high tier final review, is fixed
   by the tier. This qualifies Part 5 items 3 and 7: the tier still never rises above high
   after Step 4, but the xhigh Step 5 model is resolved from the diff. The rule is written
   down because an "or" cell in the table with no rule would let two runs of the same task
   pick differently, and the run log could not say why. Superseded by Part 10 item 4: the
   xhigh cell is fixed, and the final review cell at high follows the trigger.
4. **At high tier and above, the built-in `code-review` skill reviews the diff beside
   Codex.** Its level is medium at high, high at xhigh, and xhigh at max. It is a fixed
   slot: not a fallback, never swapped, and untouched by `--no-codex`. A required pass
   that cannot run ends the run in `blocked` rather than counting as clean. The level is
   always passed explicitly, because the skill reuses the last typed level otherwise.
   `--comment` is never passed, because no PR exists at Step 5 and comments are
   ask-first, and `--fix` is never passed, because fixes go through the implementer.
   Its availability is checked only when a stage needs it, so low, medium, and plan-only
   runs gain no prerequisite. Qualified by Part 8 item 2: in Multi-repo mode the skill
   covers the primary only, and an Opus subagent fills the Claude slot for each
   additional repository. Superseded by Part 10 item 3: it runs at every tier, at the
   levels that item gives, and low and medium runs need it when Step 5 starts.
5. **A Step 5 round at high tier and above is both passes over the same diff, under one
   shared cap of 3.** This qualifies Part 2 item 10. The Codex thread keeps `--resume`;
   the Claude pass is fresh each round, since the skill keeps no thread, and a finding it
   repeats keeps its recorded disposition unless it cites new evidence. The Opus subagent
   of Part 8 item 2 is the one Claude pass that is continued. Findings from both
   are merged into one list, with the source kept, and fixed in one batch, so a round
   costs one implementer pass, not two. Both passes must return before the round ends;
   otherwise "cap of 3" could mean three rounds per reviewer. Superseded by Part 10
   item 3 for the tier scope: a round is both passes at every tier.
6. **Step 5 has no post-cap orchestrator fix.** Step 5.3 used to allow one, while Step 5.4
   required every Step 5 fix to be seen by a later round. The two rules conflicted. The
   post-cap fix stays in Step 4 only; a Step 5 blocking finding open after the cap ends the
   run in `blocked`. The same reasoning bounds the last round: a fix made in the third
   round would have no round to review it, so the third round fixes nothing. A blocking
   finding there ends the run in `blocked`, and a non-blocking one is deferred. Two rounds
   fix, the third confirms.
7. **CI repair and Step 6 repair use the paired round at medium tier and above.**
   Medium enters the Codex slot; high and above also rerun the Claude pass fresh. Each CI
   cycle keeps its one extra Step 5 round, which now holds both passes. Low tier keeps the
   orchestrator's own review. Superseded by Part 10 item 6: every tier uses the paired
   round.
8. **The fallback follows the Codex model.** `gpt-6-sol` falls back to Opus, and
   `gpt-6-astra` to Fable then Opus, at any tier. This qualifies Part 4 item 9, which
   named the max tier because only max used `gpt-6-astra` then. Qualified by Part 10
   item 10: for reviewer calls only.
9. **Every Claude pass targets the base commit.** Read from the installed Claude Code
   2.1.284 prompt: with no target the skill diffs `@{upstream}...HEAD`, else
   `main...HEAD`, else `HEAD~1`, and adds `git diff HEAD` for uncommitted work. The work
   branch has no upstream before the push, and the loop fetches the remote default branch
   without moving local `main`, so a stale local `main` would put unrelated commits under
   Claude's review while Codex reviews only the task. That breaks the shared-diff rule and
   can produce blocking findings the task did not cause. Passing the base commit as the
   target on every pass, not only in CI repair, pins both reviewers to the same range.
   Whether the skill honors a commit as its target is unverified; acceptance item 58
   checks it with local `main` deliberately behind the base. Superseded by Part 9 item 1:
   a bare commit target reviews only that commit, so the target is the range.

## Part 8: Live-run amendments, 2026-09-29

Eight issues came from live runs of 2026-09-29. Each rule below has the reason and the
run that taught it.

1. **Step 0 detects the remote host, and a non-GitHub host ends in `prepared`.** The run
   selects the remote (the one `gh repo view` resolves when it succeeds, else `origin`,
   else the only remote), reads its URL from `git remote -v`, and classifies the host as
   `github` when `gh repo view` succeeds (GitHub Enterprise included). When it fails and
   the URL host is `github.com`, the command rejects the request because `gh` is not
   authenticated. When it fails and the host is anything else, the host is `other`. An
   unauthenticated `gh` fails the same way on an Azure remote as on a GitHub Enterprise
   remote, so the command cannot tell them apart, and a GitHub Enterprise checkout needs
   a `gh` login for that host. On `other` it accepts only file and text inputs, runs
   Steps 0 to 6, never runs Step 7, and hands publication to the repo's own
   tooling. Step 0 and Step 7 are written around `gh` and GitHub Actions, so on
   another host they cannot run as written. A full Azure DevOps path would duplicate
   Step 7.3's CI rules for a second API with no way to verify them here, so it is out of
   scope. Observed on a live run of 2026-09-29 (#4).
2. **Multi-repo mode: `--repo <path>` names additional writable checkouts.** Each
   repository gets its own base commit, branch, baseline, checks, and PR, and artifacts
   live in the primary's `.ccl/<run-id>/`. No slice spans repositories, which extends the
   no-shared-file rule. A bare `#n` is always the primary's issue, so it stays
   unambiguous. An issue closes only through the PR in its own repository, and the other
   PRs cite it with `Refs <owner>/<repo>#n`. Codex `review` reads only the session's
   checkout, so an additional repository is reviewed through `ask` with a patch file,
   and the `code-review` skill covers the primary only. Every `gh` call for an additional
   repository is run from that checkout or targeted with `-R <owner>/<repo>` where the
   subcommand accepts it (`gh api` does not, so its endpoint is spelled out), and every
   `git` call uses `git -C <path>`, so no call lands on the wrong repository. The
   primary's `.ccl.json` governs `commit` and `timeouts`, and each repository's own
   `checks` list is read for that repository. The
   operator generalized all of this by hand across three writable repositories, with no
   rules to follow. Observed on a live run of 2026-09-29 (#5).
3. **`prepared` is a terminal state, and `--no-publish` withholds Step 7.** Every step
   through Step 6 is done, no blocking defect is open, and Step 7 was withheld before any
   push, by the flag, by a non-GitHub host, or by a denied or unconfirmed Step 7 action.
   This qualifies carve-out 3: a denial before any push ends in `prepared`, and a denial
   after a push, or in Steps 0 to 6, still ends in `blocked`. `blocked` reads as a
   failure and `stopped` assumes no branch and a fresh rerun, so neither fits finished,
   unpublished work. The report gives the branch, the commit state, and the commands to
   publish. Observed on a live run of 2026-09-29 (#6).
4. **Codex availability does not depend on the session's skill list.** It is decided from
   `codex --version` and the installed codex-lite version, 0.7.0 or later (0.8.0 since
   Part 10 item 13). A Skill call that errors because the skill is not listed counts as a
   `failed` call: retry once, then swap, recording "skill not listed in session", and
   Codex is recorded unavailable for the rest of the run so later stages do not repeat the
   failed calls. The host, not the plugin, controls which skills a session lists, and with
   the CLI ban a listing gap left no route to Codex. Observed on a live run of 2026-09-29
   (#7).
5. **Implementers match the repository's line endings, and Step 4.3 checks them.** A new
   file takes the `eol=` attribute when one applies; under `text` or `text=auto` with no
   `eol=`, git normalizes on commit and no ending is enforced; else the majority of files
   in its directory, else the majority of tracked files. An edited file keeps its own. In a
   CRLF repository, LF files made git warn on every diff. Observed on a live run of
   2026-09-29 (#8).
6. **Independent slices run as one Workflow or as parallel Agent calls.** The orchestrator
   chooses, defaults to Agent calls when review rounds are expected, and logs the choice
   in `run.md`. Workflow agents cannot be continued for review rounds, and continuation
   was the more useful property. Qualified by Part 10 item 8: Codex slices run in series
   and never inside a Workflow. The issue cited `docs/decisions.md` items 97 to 99,
   which do not exist; the rule it means is Part 3 item 8. Observed on a live run of
   2026-09-29 (#9).
7. **The run budget default is by tier, and it can be extended per run.** Low and medium
   get 120 minutes, high 240, xhigh and max 360. The precedence is `--run-budget`, else
   `.ccl.json` `timeouts.run`, else the tier default. An explicit value is never replaced
   by a tier default, and 240 applies only until Step 1.5 sets the tier when nothing is
   explicit; a Step 4.5 rise to high moves an implicit budget to the high default. An
   explicit instruction in the session replaces the budget from that point.
   A flat 240 would have ended a large max run in `blocked` against the user's
   instruction to finish. Observed on a live run of 2026-09-29 (#10).
8. **A dropped call is not a denial, and a clean tree with skip-worktree edits runs in a
   detached worktree.** A dropped call returns no result and no explicit
   denial; a result that names a hook, a permission rule, or the permission mode as the
   reason is a denial. A dropped read-only call is retried once, serially. A dropped write
   is checked
   first, and retried once only if it did not take effect. Carve-out 3 applies only to an
   explicit denial, so a drop no longer ends a run in `blocked`. Git hides skip-worktree
   and assume-unchanged edits from `git status --porcelain`, so the tree reads clean.
   When status and index are clean and at least one path marked `S`, `h`, or `s` in
   `git ls-files -v` differs from `HEAD` (compared through `git cat-file --filters`, so
   line-ending conversion is not read as an edit), Step 0.3 creates a detached worktree
   from the base commit as the run's checkout. The plugin does not stash, and a reset was denied as
   destructive on the live run. This is a narrow use of the deferred `--worktree`
   feature, not the feature. The run installs the repository's dependencies in the
   worktree before the baseline when the instruction files or a lockfile name an install
   step, else a check that needs them is recorded as not run, with the reason. Observed
   on a live run of 2026-09-29 (#11).

Composition rules, each with its reason:

- **A worktree run is not allowed at high tier or above.** The `code-review` skill reviews
  only the session's checkout, so it cannot review the worktree. A run that reaches high
  at Step 1.5, or is raised to high at Step 4.5, ends in `blocked` naming the skip-worktree
  files that differ from `HEAD` and the tier. Superseded by Part 10 item 5: a worktree
  run is allowed at every tier, with the Opus subagent as its Claude slot.
- **A worktree run and Multi-repo mode do not combine.** The primary's `codex-lite:review`
  and `code-review` read the session's checkout, so a primary in a worktree would be
  reviewed wrongly. A primary that would qualify ends in `blocked`, naming the
  skip-worktree files that differ from `HEAD`. An additional repository does not block;
  it continues (Part 9 item 3).
- **Every repository is on the same host as the primary, by hostname.** One run has one
  publish path, and `-R <owner>/<repo>` without a host resolves on github.com, so a GitHub
  Enterprise checkout beside a github.com primary is a different host. It is a preflight
  failure, and on `other` all repositories end in
  `prepared` together.
- **An additional repository's Claude slot is an Opus subagent.** Part 10 item 5 uses
  the same substitute for a worktree run. The `code-review` skill cannot target a checkout
  other than the session's. The subagent is a defined substitute, recorded in `run.md` and
  named in the report. It is not a swap, and it is continued, not fresh, so that its
  follow-up rounds keep their context.
- **Multi-repo mode with `"commit": true` commits the snapshot in one repository.** It
  goes to the first repository, the primary first and then the `--repo` order, that has a
  diff. A repository with no diff never receives it and gets no PR.
- **`--no-publish` with `"commit": true` commits nothing.** Step 7 never runs, so no
  commit is made and the report gives the commit commands. A denied push with
  `"commit": true` leaves a commit that carries the `publishing` snapshot, and the report
  says so.

## Part 9: 0.6.0, 2026-09-30

Six issues from live runs and reviews of 2026-09-29 and 2026-09-30, one commit each on
the 0.6.0 branch. Each rule below has the reason, the issue, and the test or run that
taught it where there is one.

1. **Every Claude `code-review` pass takes the level and then `<base-commit>...HEAD`.**
   This supersedes Part 7 item 9. The skill's own diff step is prompt text that the model
   follows, so the call shape was settled by running the skill through the Skill tool,
   the path the orchestrator uses, on 2026-09-30 (Claude Code 2.1.284) in a throwaway
   repo whose local `main` was two commits behind the base. A bare commit target made the
   skill run `git show <base>`, which reviews that commit only and misses all the
   uncommitted work; this reproduced the bug. No target, with the upstream pinned to a
   local branch at the base, was followed in one run and ignored in another (it ran
   `main...HEAD` and put the stale commits under review). The range target was correct in
   five of five runs, at low, medium, high, and xhigh, in the Step 5 state and the CI
   repair state: each ran `git diff <base>...HEAD` and `git diff HEAD`, which together are
   the base to the working tree. The direct test chose the shape because acceptance item
   58 is a full run, and the loop runs from the installed plugin, so item 58 can only pass
   after 0.6.0 is installed. Item 58 stays the post-install check. Issue #15 said that
   tracking `<remote>/<default>` holds only while nothing fetches the default branch. That
   is wrong for a three-dot range, which diffs from the merge base, and a fetch does not
   move the merge base. Tracking was rejected anyway. For an existing pushed branch, the
   first push moves the upstream to `HEAD`, and the committed task work drops out of
   `@{upstream}...HEAD`. A bare `git push` also refuses when the upstream name differs
   from the branch name. Issue #15.
2. **`--continue <branch>` continues an existing remote branch; a pull request is still
   not an input.** The user chose it on 2026-09-30 over `--base <ref>` and over accepting
   a PR reference, because it keeps the input guard as it is and finds the PR from the
   branch. The base is the remote branch head, fetched at Step 0.2, so the diff is only
   this run's work. The PR body belongs to the PR's author, so the run posts its filled
   `pr-body.md` as one comment and never edits the body. The comment cannot close an
   issue, so it carries no closing keyword: it lists each issue with its status and says
   that the PR's own body decides what merging closes. A local branch of that name that
   differs from the remote head is a preflight failure, because the run never resets
   local work. The remote head is checked before the first push and before each CI repair
   push, against the base commit and the last commit this run pushed, and a move ends the
   run in `blocked`, since the run never forces a push. In Multi-repo mode the branch must
   exist on the primary's remote; each additional repository continues it where it exists
   and creates it where it does not, so one branch name serves all of them. Repair mode
   stays out, for the reason in Part 2 item 9: the run reads no review comments and no CI
   state from before the run. Review of the branch added five rules. Every remote branch
   check queries `refs/heads/<branch>`, because `git ls-remote --heads <remote> <branch>`
   with a bare name matches any ref whose path ends in it (reproduced on 2026-09-30), and
   this covers the Step 3.7.2 new-branch collision check that predates `--continue`. The
   flag value is rejected when it starts with `-`, has a character other than letters,
   digits, `.`, `_`, `/`, and `-`, or fails `git check-ref-format --branch`, so it cannot
   act as an option or a revision expression. The character list already excludes `@`
   and `{`, so a separate rule for revision syntax could never fire and was removed.
   Step 0.2 fails a branch that another worktree has checked out, because `git switch`
   refuses it. With `--continue`, `HEAD` of the session's checkout must be at the base
   commit, on the branch or detached at it, or Step 0.2 fails with the command to run:
   `git merge --ff-only <remote>/<branch>` when the session is on the branch and it is
   behind, because `git switch <branch>` then does nothing; `git switch <branch>` when
   the session is elsewhere and the local branch equals the remote or is absent; else
   `git switch --detach <remote>/<branch>`. The pull request list uses
   `isCrossRepository`, because an owner field also passes a fork with the same
   owner. An earlier draft read the base commit with `git show` when `HEAD`
   differed, but the Codex plan review and the Step 1.2 reproduction read the session's
   checkout, so they checked the wrong code. The requirement leaves the planning snapshot
   rule unchanged. A session on the branch blocks a worktree run, and one detached at the
   base commit does not. A plan-only run records a differing local branch, a branch
   checked out in another worktree, and the pull request cases instead of failing,
   because they matter only for building and publishing; the `HEAD` requirement stays,
   since it makes the plan correct. A `prepared` run with an open PR gives the `gh pr
   comment` command with the absolute path of a body file, written at that point if Step
   7.2 did not, one per repository at its own path in Multi-repo mode, so the user can
   finish the run by hand. Every `--body-file` is an absolute path, because in a worktree
   run the `gh` call runs inside the checkout. A later review added four rules. The
   default branch is rejected as the value, because the run would push straight to it.
   The branch is fetched with the explicit refspec
   `+refs/heads/<branch>:refs/remotes/<remote>/<branch>`, because a plain `git fetch
   <remote> <branch>` updates `<remote>/<branch>` only when the fetch refspec covers it,
   which a single-branch clone does not. A plan-only run needs `HEAD` at the remote head
   too, so its relaxation for a differing local branch applies only with `HEAD`
   detached there. A reverification that needs a revision with no Step 3 round left ends
   in `blocked`. Every artifact path passed as an argument to a shell command is absolute;
   request text for Codex keeps naming files relative to the session's checkout. A
   further review added five rules. A repository whose remote lacks the branch creates it
   under the `continue` name, with the collision check, because the naming rule would
   derive a different name. A PR the run opens for a continued branch says how many
   earlier commits of the branch the run did not review, because its body covers only
   this run's diff. A `--no-publish` run records the closed, merged, and several-open
   pull request cases instead of failing, because it never publishes; its local branch
   and worktree failures stay, because it still switches and commits. The pull request
   list takes `--limit 100`, because `gh` returns 30 by default. It reads open pull
   requests first and the rest only when none from this repository is open, because
   `--head` also matches forks and a popular branch name can fill 100 results with
   them; a result of 100 that does not settle the case is a failure rather than a
   guess, and it excludes the other cases. The pull requests are read again after a
   `--confirm-plan` yes and before the first push, and a changed base branch ends in
   `blocked`, because CI would otherwise be judged against the old target's required
   checks; reading them after the yes stops the run before implementation rather than
   after it. The CI watch reads the PR at every poll and blocks on a retarget for every
   PR, including one the run opened, since any PR can be retargeted while CI runs. The
   flag value also rejects shell metacharacters, because it is placed into many shell
   commands. Issue #16.
3. **The run never adopts Multi-repo mode from prose, and an additional repository with
   skip-worktree edits continues.** A task that named other writable repositories in prose
   ran with `repos` set to `none` and no rule. Step 1 now ends in `blocked` before Step 2,
   and the report gives the rerun command with one `--repo <path>` per repository, because
   guessing the set of writable checkouts from text could edit the wrong one. An
   additional repository whose skip-worktree or assume-unchanged files differ from `HEAD`
   had no rule. It continues, the paths are recorded in `run.md`, and the report names
   them as local state its baseline and Step 6 checks ran against. Its review is not
   affected, because its patch comes from `git -C <path> diff <base>`, which leaves those
   paths out. No slice may edit such a path: an edit to it is invisible to `git diff`,
   `git add -N`, and staging, so review and the commit would drop it while local checks
   pass on it, and a plan that needs one ends in `blocked` at Step 2. Step 1 blocks on
   any writable checkout that is not the primary and not
   in `repos`, because a run that lists one of two needed checkouts would otherwise edit
   only the listed one; the rerun command adds one `--repo` for each missing checkout.
   In Multi-repo mode each repository's PR body is its own file,
   `.ccl/<run-id>/pr-body-<slug>.md` in the primary, passed by absolute path to every
   `gh` body call for an additional repository. The call runs inside that repository's
   checkout, where `.ccl/<run-id>/` does not exist, so the relative path that the
   `gh pr create` rule had before 0.6.0 was wrong. Issue #18.
4. **Times in `run.md` come from a narrower `date` rule, not a helper script.** The user
   chose this on 2026-09-30. The old rule asked for `date` before every step and every
   call, and on a 3.3 hour live run the logged times drifted and had to be corrected. The
   run budget is enforced only at step boundaries and around timed calls, so `date` runs
   there, right after each push returns, at the start of Step 0, and at the terminal
   state, and nowhere else, so the 2 minute CI wait and the CI budget count from the push.
   A helper script is a new kind of file in a prompt-only plugin, and it needs an
   `allowed-tools` entry and testing in both Git Bash and PowerShell. The worktree install
   step is a timed call under the check budget, so it gets the same `date` and budget
   comparison as a check. One format, `date -u +%Y-%m-%dT%H:%M:%SZ`, is used for the whole
   run, and every logged time is copied from the command's output, never recalled or
   derived from earlier entries. Elapsed time is the Step 0 start to the latest recorded
   time, less each Step 3.5 wait (item 5), so a wait cannot count against the budget. The
   skill no longer carries the 2026-09-30 test note for the range target; item 1 is the
   record. Issue #19.
5. **`--confirm-plan` pauses once for approval after the plan is final.** A task that
   said to plan first and then approve had no supported path: `--plan-only` ends the run,
   and a later run starts over from Step 0. Step 3.5 runs the Step 3.7.1 reverification
   before it asks, so the user approves the plan that would be built and not one the
   reverification then revises. The wait does not count against the run budget, because
   the live run that asked for this did the same by hand and a long wait would otherwise
   end a run in `blocked` for time the user spent. A requested change is another Step 3
   round, inside the cap of 3, that follows Step 3 items 2 to 6, so an open blocking
   objection at the cap ends `blocked` and the question is asked again only after a round
   with none; when no round is left the run ends `stopped` and the report gives the
   requested change. After the approval, before Step 3.7.2, the clean-tree check
   reruns, and with `--continue` so do the check that the local branch equals the base,
   the worktree check, and the `HEAD` check, because the user can change the tree or the
   checkout during a long wait; a failure ends in `blocked`. The skip-worktree
   comparison reruns too, because an edit hidden during the wait would pass the other
   checks and reach local checks without review; a differing path that Step 0.3 did not
   record fails rather than starting the worktree exception mid-run.
   Any reply that is not a clear yes ends in `plan-only`, so an unclear answer never
   starts an implementation. The flag is rejected with `--plan-only`, and `/ccl:plan`
   rejects it as not applicable. Issue #17.
6. **Rarely used paths live in their own files, read only when the run takes the path.**
   The worktree run is in `worktree.md`, Multi-repo mode in `multi-repo.md`, and the CI
   watch details (Step 7.3 items 1 to 4) in `ci-watch.md`. SKILL.md keeps a pointer at
   each place. The non-GitHub host paragraph stayed, because it is about six lines and a
   pointer costs about as much. The move commit changes no rule: every removed line is in
   a new file verbatim, and the only new text is the pointers and the cross-references
   that named the old sections. Later 0.6.0 commits changed rules in these files, and
   items 2 and 3 list them. On a default single-repository GitHub run, the lines read
   through Step 6 (SKILL.md and `tiers.md`) were 1,186 at 0.5.1, 1,309 before the move on
   this branch, and 1,115 after. In total, adding `report.md`, `pr-body.md`, and after
   the move `ci-watch.md`, they were 1,409 at 0.5.1, 1,553 before the move, and 1,431
   after. Later edits change these counts; acceptance item 100 pins them to the move
   commit. Issue #20.

## Part 10: 0.7.0, Codex implementers and review at every tier, 2026-09-30

Two goals: spend Claude weekly usage on orchestration and review rather than on
implementation, with Codex staying the primary adversarial reviewer, and give every tier,
low included, a final review by both a Codex reviewer and the Claude `code-review` skill.
ccl 0.7.0 needs codex-lite 0.8.0 for the `implement` command. No incident is recorded for
these rules; each has its reason.

1. **Codex implements every slice by default, one model per tier.** The model is
   `gpt-6-luna` at low, `gpt-6.1-sol` at medium and high, and `gpt-6-astra` at xhigh and
   max, called through `codex-lite:implement`. The orchestrator still reviews each slice
   in Step 4. This supersedes Part 6 item 3, which had Sonnet as the default. Claude usage
   then goes to orchestration and review, and a cheaper model implements where a slice
   does not need Opus. The implementer is recorded in the plan per slice at every tier,
   as "codex" or the Opus criterion, and the slice's effective model is recorded as
   before.
2. **Opus replaces the Codex implementer at high, xhigh, and max only by the existing
   criteria.** The criteria are a risk floor trigger, more than eight files, or a new
   module, type, interface, or rule section that another file cites (Part 6 item 3). They
   do not apply at low and medium. Sonnet is never chosen at the plan. It is the
   fallback: under `--no-codex`, when Codex was found unavailable at Step 0.6, when a
   Codex implementer call returns `failed` or no status line twice in a row, and when an
   `opus` call errors (Part 6 item 4). Choosing Sonnet at the plan would put a third
   default beside the Codex one, and the criteria already say when a slice is hard.
3. **The Claude `code-review` skill runs at every tier, and low tier has a final review.**
   The levels are `low` at low, `medium` at medium, `high` at high with a trigger and
   `medium` at high without one, `high` at xhigh, and `xhigh` at max. This supersedes Part
   7 item 2, which let low skip Step 5, and Part 7 items 4 and 5, which limited the Claude
   pass and the paired round to high tier and above. Low was the only tier with no diff
   review. It also ends Part 7 item 4's claim that low, medium, and plan-only runs gain no
   prerequisite: low and medium runs now need the skill when Step 5 starts, and a missing
   skill ends them in `blocked` there. A plan-only run still needs nothing. The low and
   medium run budgets of 120 minutes now include Step 5.
4. **Only the high cell follows the trigger.** The plan review at high uses `gpt-6-astra`
   with a trigger, judged at the Step 1.5 floor check, and `gpt-6.1-sol` otherwise, as
   before. The final review at high uses `gpt-6-astra` and `code-review high` when a
   trigger was present at the estimate or is in the diff after Step 4, and `gpt-6.1-sol`
   and `code-review medium` otherwise. A medium run that rises to high always has a
   trigger in the diff, so it gets the trigger cell. The xhigh and max cells are fixed.
   This supersedes Part 7 item 3, whose xhigh final review followed the trigger, and
   qualifies Part 5 items 3 and 7, whose floored high run kept a `gpt-6.1-sol` diff review
   and whose resolved Step 5 model made xhigh follow the diff. Each cell that follows a
   rule has the rule written down, for the reason in Part 7 item 3.
5. **A worktree run is allowed at every tier.** A worktree run at any tier uses the Opus
   subagent substitute that `multi-repo.md` defines for additional repositories as its
   Claude slot, because the skill reviews only the session's checkout. This supersedes the
   Part 8 composition rule that refused a worktree run at high tier and above, and the
   gates at Step 0.3, Step 1.5, and Step 4.5 that enforced it. The substitute is a
   defined one, recorded in `run.md` and named in the report, not a swap. The rule that
   a worktree run and Multi-repo mode do not combine stands.
6. **A Step 6 check-failure fix and a CI repair go through a Step 5 round at every tier.**
   Every reviewer the stage has reads the fix, within Step 5's cap. A CI repair uses the
   Step 5 Codex thread when one exists, else `codex-lite:review --base <base-commit>`,
   plus the Claude reviewer, and each cycle keeps its one extra review round. This
   supersedes Part 7 item 7 and the low-tier part of Part 3 item 11, which left low with
   the orchestrator's own review. Low now has a Step 5, so nothing is left to bypass. An
   ordinary Step 6 repair stays inside Step 5's cap and ends in `blocked` when that cap is
   exhausted.
7. **A Codex implementer thread is not resumed, so every fix round is a fresh call.**
   Resume is read-only in codex-lite, and `implement` has no `--resume`. Each Step 4.3 or
   Step 5.3 fix round for a Codex slice is a new `implement` call given the findings and
   the slice's current diff, as a replaced agent is today. The thread of an earlier call
   can be questioned read-only with `ask --resume`, but it is not used to write. This
   qualifies Part 3 item 8 for Codex slices. `SendMessage` continues Opus and Sonnet
   implementers only.
8. **Codex slices run in series, never inside a Workflow.** codex-lite has one request
   file and one thread file per session, so Codex calls stay one at a time, and that now
   includes implementers. Independent Codex slices therefore run one after another. Opus
   slices and Sonnet fallbacks may still run in parallel with each other. This qualifies
   Part 8 item 6.
9. **The orchestrator installs dependencies, because Codex has no network.** A dependency
   the plan adds is installed in Step 3.7, after the baseline checks of Step 3.7.3 ran on
   the unchanged base and after the ask-first rule, before any implementer starts. The
   manifest and lockfile edits are part of the run's diff and are reviewed in Step 5 like
   any other change. A slice's checks that need the network are run by the orchestrator
   after the implementer returns, and a failure goes back to the implementer as a finding.
   The implementer choice does not change because of this.
10. **Two budgets and two fallback chains, keyed by role.** Reviewer Codex calls keep the
    Codex budget, capped at 60 minutes, and the reviewer fallbacks, `opus` for
    `gpt-6.1-sol` and `fable` then `opus` for `gpt-6-astra`. A Codex implementer call
    takes the smaller of the subagent budget and the remaining run budget, capped at 3600
    seconds because codex-lite refuses a larger `--timeout`, and falls back to `sonnet`
    only. The cap and the value passed are logged in `run.md`; a `timeouts.subagent`
    above 60 minutes is passed as 3600. Opus and Sonnet Agent implementers keep the full
    subagent budget. This qualifies Part 4 item 9 and Part 7 item 8, whose fallback
    followed the Codex model without regard to the role. The rules that said "every Codex
    call" carries the Codex budget, and "a `gpt-6-astra` stage" falls back to `fable`, now
    apply to reviewer calls only. A `refused` status from an implementer ends the run in
    `blocked`, as for a reviewer, and a `timeout` status is a budget expiry.
11. **A failed Codex implementer call may have written part of the slice, so the retry
    waits for the process to end.** Before a retry or the Sonnet fallback, the previous
    call must have returned its output, in the foreground or as a background completion
    notification, because codex-lite ends the Codex process when its turn ends or the Bash
    call times out. A call whose output never arrives is a budget expiry: the run ends in
    `blocked` and no other writer starts on that checkout. Output that says Codex may
    still be running has the same result: codex-lite prints "codex may still be running as
    pid" when the process outlived its hard end, and on Windows warns that child
    processes may still be running after a timeout. A `failed` result on Windows carries
    no such warning even though a command Codex started may outlive it, and nothing the
    orchestrator can read proves the process tree is gone, and a result with no status
    line was cut off before the runner could warn at all. On Windows a Codex implementer
    call that returns `failed` or no status line is therefore not retried and gets no
    Sonnet fallback: the run ends in `blocked` naming the possible surviving process. On POSIX the runner stops the process
    group, so the returned output is the evidence. The orchestrator then reads the tree
    state from the footer or `git status`, and gives the next call the current diff with
    the same slice prompt. This applies the dropped-write rule (Part 8 item 8, Approval
    scope item 6) to implementers, and the threshold stays two failures in a row.
12. **The `implement` command weakens one codex-lite guarantee, and the loop accepts
    it.** Claude can invoke `implement`, so writes are no longer gated on a typed
    command. The codex-lite README documents the tradeoff, and the command's description
    says when Claude may invoke it: only when the user, or a skill the user invoked,
    delegates a change to Codex. In a session in default permission mode the Skill call
    still prompts. ccl invokes it only for a slice of a run the user started.
13. **ccl requires codex-lite 0.8.0 or later.** This supersedes the 0.7.0 minimum of Part
    3 item 18, Part 4 item 3, and Part 8 item 4, because 0.7.0 has no `implement`. The
    call is `codex-lite:implement` with `--model`, `--timeout` (1 to 3600), and `--cwd`
    last, since its value is the rest of its line and the request text starts on the next
    line. Its output lines are those of `do`, including the `status:` line and the tree
    footer. A worktree run, or an additional repository in Multi-repo mode, passes
    `--cwd <checkout>`, and its request names files by absolute path or carries the
    content inline, because Codex runs in that checkout and the rule that requests name
    files relative to the session's checkout applies to reviewer calls only. A Skill
    call for `implement` that errors because the skill is not listed follows Part 8
    item 4.

## Rules stated elsewhere in the loop, with reasons

These are not numbered decisions, but the same reasoning applies.

- **Codex model ids are always the full id.** Bare `sol` fails on a ChatGPT account.
- **Every Codex reviewer call carries `--timeout`, and follow-ups pass `--resume <id>`,
  never bare.** Step 3 and Step 5 are separate threads. A bare `--resume` after a plan
  review would continue the wrong one. An implementer call carries `--timeout` from the
  subagent budget and has no `--resume` (Part 10 items 7 and 10).
- **Only the orchestrator calls Codex, one call at a time.** codex-lite supports one
  call at a time per Claude session, so Codex implementer slices run in series too
  (Part 10 item 8).
- **A fallback swaps one reviewer or one implementer and never removes a stage.** The
  tier is set by the task's risk and does not change because a reviewer is unavailable.
- **A shell quoting failure is fixed by moving the text into a file, not by
  requoting.**
- **A subagent report is model output, not user approval.** It cannot grant anything.
- **A denied permission is never retried or routed around.** Routing around a denial
  would turn a control into a suggestion. A dropped call is not a denial (Part 8 item 8),
  and a denied Step 7 action before any push ends in `prepared` (Part 8 item 3).
