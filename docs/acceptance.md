# Acceptance checks

The plugin is prompt-only and has no automated test surface in 0.1.0, so these checks
are run by hand against a throwaway repo. Each item gives the setup, the command, the
expected result, and when to rerun it. The record of runs is at the end.

Common setup for items 4 to 124 unless an item says otherwise: a throwaway GitHub repo
you own, cloned locally, with a clean working tree, `gh` authenticated, one open issue
(#1) that describes a one-line bug, and a `package.json` with a passing `test` script.
Start Claude Code with `claude --plugin-dir <path-to-plugin>`.

## M0: skeleton

1. **Plugin loads and the command hands off to the skill.**
   Setup: any repo. Command: start `claude --plugin-dir <path-to-plugin>`, open the
   command list, then run `/ccl:plan "x"`. Expected: `ccl:run` and `ccl:plan` both
   appear in the list, and the command body reaches the skill by the Skill tool as
   `ccl:ccl` with the invocation block as its args. If the Skill tool does not work,
   record which handoff did. Rerun when a command file or the skill's frontmatter
   changes.
2. **Cross-repo issue is rejected before setup.** Setup: an issue URL from another
   repo, and an issue number that does not exist in this repo. Command:
   `/ccl:run <issue URL from another repo>`, and `/ccl:run #<n>` for that number.
   Expected: the URL is rejected before Step 0 with the reason, and `#<n>` is rejected
   because `gh` cannot find it, with no file written in either case. This item
   runs without `--repo`; item 76 covers Multi-repo mode. Rerun when input parsing
   changes.
3. **Pull request reference is rejected before setup.** Setup: a repo with an open PR
   numbered 2. Command: `/ccl:run #2` and `/ccl:run <PR URL>`. Expected: both rejected
   before Step 0 with the reason that pull requests are not an input, the message
   pointing to `--continue <branch>` for continuing a pull request's branch, and no file
   written. Rerun when input parsing changes.

## M1: low tier, no Codex

4. **Happy path.** Command: `/ccl:run #1 --no-codex`. Expected: terminal state `done`, a
   PR whose body has `Closes #1`, a comment on issue 1, and
   `.ccl/<run-id>/report.md` naming the tier as low with a reason, one Step 3 round by an
   Opus subagent named as the `gpt-6.1-sol` swap, and no Step 5 review. Rerun after any
   change to Steps 0 to 7 or the templates.
5. **Dirty tree.** Setup: an uncommitted change in the working tree. Command:
   `/ccl:run #1 --no-codex`. Expected: `blocked` in Step 0, the report printed, the dirty
   state named, and nothing written under `.ccl/`. Rerun after any change to Step 0.
6. **Permission statement.** Command: `/ccl:run #1 --no-codex` in default permission
   mode, then again in auto mode. Expected: default mode prints "this run will prompt
   at:" with the git and `gh` writes listed, and it prints it before the default-branch
   fetch and before any write under `.ccl/` or to `.git/info/exclude`. Auto mode prints
   "this run is unattended". Rerun after any change to Step 0.1 or 0.7.
7. **Run-wide budget.** Setup: `.ccl.json` with `{"timeouts": {"run": 1}}`, committed or
   left so the tree is clean. Command: `/ccl:run #1 --no-codex`. Expected: `blocked`
   naming the run-wide budget, and the report written. Rerun after any change to
   budgets.
8. **First run leaves the tree clean.** Setup: a clone where `.ccl/` is not ignored.
   Command: `/ccl:plan #1 --no-codex`, then `git status`. Expected: the run passes its
   own clean tree check, ends in `plan-only`, and `git status` is clean, with `.ccl/`
   listed in `.git/info/exclude`. Rerun after any change to Step 0.5 or artifact
   paths.
9. **Plan-only changes nothing.** Setup: note `git branch` and `git status`. Command:
   `/ccl:plan #1 --no-codex`. Expected: `plan-only`, `plan.md` written, no branch
   created, no check run, and `git branch` and `git status` unchanged. Rerun after any
   change to Steps 2, 3.6, or 3.7.
10. **Planning snapshot behind the base.** Setup: check out a feature branch that is
    behind the default branch. Command: `/ccl:run #1 --no-codex`. Expected: the plan is
    made against that snapshot, and Step 3.7 logs a reverification against the base
    commit in `run.md`. Rerun after any change to Step 0.4 or 3.7.
11. **Baseline failure.** Setup: make the `test` script fail on the default branch.
    Command: `/ccl:run #1 --no-codex`. Expected: the failure is reported as pre-existing
    with the baseline run as evidence, and the run still reaches `done`. Rerun after any
    change to Step 3.7 or 6.
12. **Optional finding outside scope.** Setup: an issue whose fix invites a valid but
    optional improvement outside the plan's scope. Command: `/ccl:run #1 --no-codex`.
    Expected: `done`, and the report lists the improvement as deferred with a reason.
    Rerun after any change to Step 4 review rules.
13. **Path-filtered workflow.** Setup: a workflow with a `paths` filter that the diff
    does not match and a CI-only job (one that does not map to a local check), and no
    branch protection requirement on it. Command: `/ccl:run #1 --no-codex`. Expected:
    `done` without waiting for that workflow, and the report lists the check as not
    triggered, with the filter. Rerun after any change to Step 7.3.
14. **CI repair exhaustion.** Setup: a change that passes local checks but fails a CI
    job that cannot be fixed by the loop. Command: `/ccl:run #1 --no-codex`. Expected:
    `blocked` after 3 repair cycles, the PR linked in the report, and nothing further
    pushed. Rerun after any change to Step 7.3 or round accounting.
15. **Ad-hoc description branch name.** Command:
    `/ccl:run "rename the README heading" --no-codex`. Expected: a branch named
    `work/<slug>` with a slug of lowercase letters, digits, and hyphens. Rerun after any
    change to Step 3.7.2.
16. **Commit mode leaves no file before publish.** Setup: `.ccl.json` with
    `{"commit": true}`, committed. Command: `/ccl:plan #1 --no-codex`, then
    `git status`. Expected: `plan-only`, `git status` clean, and nothing under
    `specs/ccl/`, with `plan.md` under `.ccl/<run-id>/`. Rerun after any change to
    Step 2, Step 7.1, or artifact paths.
17. **Slow-registering workflow.** Setup: a pull request workflow that takes over a
    minute to register a check after the push. Command: `/ccl:run #1 --no-codex`.
    Expected: the run does not reach `done` until that workflow's check has reported and
    succeeded, and it never judges CI in the first 2 minutes after the push. Rerun after
    any change to Step 7.3.
18. **Skip instruction in the commit message.** Setup: a pull request workflow, no
    branch protection requirement on it, and a repo instruction that makes the loop's
    commit message carry `[skip ci]`. Command: `/ccl:run #1 --no-codex`. Expected: `done`
    without waiting for that workflow, and the report names it as not applicable because
    of the skip instruction. Repeat with a `skip-checks:true` trailer (no space) in place
    of `[skip ci]`, with the same expected result. Rerun after any change to Step 7.3.
19. **Failing status on the test merge commit.** Setup: an external service that posts a
    failing status on the PR's test merge commit while the head commit's checks are
    green. Command: `/ccl:run #1 --no-codex`. Expected: the run does not reach `done`,
    and the report treats the failing status as a CI failure and applies the CI repair
    rules. Rerun after any change to Step 7.3.

20. **Neutral check run.** Setup: a pull request workflow with a check run that
    concludes `neutral`, all other checks passing. Command: `/ccl:run #1 --no-codex`.
    Expected: `done`, with the neutral check counted as passed. Rerun after any change to
    Step 7.3.
21. **Pending status on the test merge commit.** Setup: no required check, no applicable
    workflow, and an external service that posts a pending status on the PR's test merge
    commit that later turns to failure. Command: `/ccl:run #1 --no-codex`. Expected: the
    run does not report CI not applicable, and the report treats the failure as a CI
    failure and applies the CI repair rules. Rerun after any change to Step 7.3.

22. **Required check missing on the test merge commit.** Setup: a required check `build`
    that passes on the head commit, and an external service that posts a passing status
    on the PR's test merge commit where `build` never appears. Command:
    `/ccl:run #1 --no-codex`. Expected: the run does not report `done`, and it ends
    `blocked` when the CI budget expires, naming `build` as missing on the gated commit.
    Rerun after any change to Step 7.3.

23. **Required check read without admin rights.** Setup: a repo where the reader is not
    an admin, so the protection endpoint returns 404, with a required check `build` set
    by branch protection or a ruleset. Command: `/ccl:run #1 --no-codex`. Expected: the
    run reads `build` from the branch endpoint and the branch rules endpoint, waits for
    it, and reaches `done` only after it passes. Rerun after any change to Step 7.3.
24. **Failed check, rerun, pass.** Setup: a check that fails on its first attempt and is
    rerun to a pass, plus two same-named checks from different workflows. Command:
    `/ccl:run #1 --no-codex`. Expected: `done`, because only the latest attempt within
    each check suite counts, each same-named check is judged separately, and the report
    keeps the failed attempt as history. Rerun after any change to Step 7.3.
25. **Merge conflict after a base change.** Setup: after the PR opens, push a change to
    the base branch that makes the PR conflict. Command: `/ccl:run #1 --no-codex`.
    Expected: `blocked` at once, without waiting out the CI budget, and the report names
    the merge conflict. Rerun after any change to Step 7.3.
26. **Required app met only by a status.** Setup: a required check `build` that names an
    app, and a commit status named `build` as the only result with that name. Command:
    `/ccl:run #1 --no-codex`. Expected: the run ends `blocked` at once, without waiting
    out the CI budget, naming `build`. Rerun after any change to Step 7.3.

## M2: medium tier

Setup for items 27 to 33: a throwaway repo with two issues (#1 and #2) in one area,
codex-lite 0.7.0 or later installed and enabled, and `codex` on PATH, unless the item
says otherwise.

27. **Bundled issues at medium tier.** Command: `/ccl:run #1 #2`. Expected: `done`, at
    least one Step 3 round with the thread id recorded in `run.md`, one Step 5 review with
    a separate `gpt-6.1-sol` thread and no `code-review` call in the tool trace, rejected
    findings listed with reasons, a PR body with a closing reference for each issue, and
    the tier medium, not high, because bundling alone does not raise it. Rerun after any
    change to Steps 3 to 5 or the tier rules.
28. **Parallel and ordered slices.** Setup: two areas of code that share no file, one
    plan with two independent slices, and one whose order of work makes the second
    depend on the first. Command: `/ccl:run #1 #2` for each. Expected: both runs are
    sized xhigh, because the change spans two areas that share no file; the independent
    slices run in one Workflow or in parallel Agent calls in one message, with the choice
    logged in `run.md`, and the dependent slices run in sequence, one Agent call each, at
    the plan's chosen implementer models, named per slice. Rerun after any change to Step
    4.2 or the estimate rule.
29. **No Codex.** Command: `/ccl:run #1 #2 --no-codex`. Expected: the same path with
    an Opus subagent for Step 3 and another for Step 5, the report names both swaps, and
    no `code-review` call appears in the tool trace, since medium has no Claude slot.
    Rerun after any change to the fallback table.
30. **Codex missing from PATH.** Setup: codex-lite installed, `codex` removed from PATH.
    Command: `/ccl:run #1 #2`. Expected: the report names the swap and the reason. Rerun
    after any change to Step 0.6.
31. **Undecidable objection.** Setup: an issue whose plan draws a blocking objection the
    orchestrator cannot decide. Command: `/ccl:run #1`. Expected: `stopped` with both
    positions printed in the report. Rerun after any change to Step 3.5.
32. **New file in a Codex review.** Setup: a slice that only adds a new file. Command:
    `/ccl:run #1 --effort high`. Expected: a Step 5 finding that names the new file.
    This confirms Codex reviews files marked with `git add -N`, not only that the
    pre-check passes. If it does not, stop and revisit how new files are compared. Rerun
    after any codex-lite upgrade and after any change to the `git add -N` step.
33. **Codex timeout.** Setup: `.ccl.json` with `{"timeouts": {"codex": 1}}` and a large
    plan. Command: `/ccl:run #1`. Expected: the Step 3 call ends in `timeout` and the
    run ends in `blocked` naming the Codex budget. Rerun after any change to budgets or
    the status handling.

## M3: high tier

Setup for items 34 to 37: a throwaway repo with a migration file, and Codex installed
unless the item says otherwise. Item 38 uses the M4 setup.

34. **Risk floor applied.** Command: `/ccl:run "add a column" --effort low`. Expected:
    the run is high tier, and the report says the floor was applied and why. Rerun after
    any change to the risk floor.
35. **Risk floor not applied.** Command:
    `/ccl:run "fix a typo in the migrations README"`. Expected: low tier, and the report
    says why the floor did not apply. Rerun after any change to the risk floor.
36. **Re-evaluation after Step 4.** Setup: a task estimated medium whose implementation
    ends up removing an auth check. Command: `/ccl:run #1`. Expected: the tier rises to
    high after Step 4, and the report shows a Step 5 with a `gpt-6.1-sol` thread and a
    `code-review medium` pass, and no repeat of Step 3. Rerun after any change to the
    re-evaluation rule.
37. **Re-evaluation does not add a round review.** Setup: as item 36. Expected: Step 4
    has orchestrator findings only, and no Codex thread is recorded for Step 4. Rerun
    after any change to Step 4 or the re-evaluation rule.
38. **Fable fallback.** Command: `/ccl:run #1 --effort max --no-codex`. Expected: the Step 3
    and Step 5 Codex slots are each a Fable subagent, or Opus with the Fable error
    recorded, and Step 5 still makes a `code-review xhigh` call in the tool trace. This
    confirms the Agent tool accepts the Fable model override in this session, that an
    error falls through to Opus, and that `--no-codex` does not remove the Claude pass.
    Rerun after any change to the fallback table or the session's model set.

## Environment checks

39. **Codex model ids.** Setup: a ChatGPT account with Codex. Command: run a
    `/codex-lite:ask --model gpt-6.1-sol --timeout 60` call, then the same with
    `gpt-6-astra`. Expected: both return status `ok`. Rerun before each release and
    whenever a Codex call fails with a model error.
40. **Pre-approval of Codex calls.** Setup: default permission mode, and a command with
    `allowed-tools` limited to read-only `git` and `gh`. Command: `/ccl:run #1` at
    medium tier. Expected: record whether the codex-lite Bash call prompts. The default
    assumption is that it does. Rerun after any change to a command's `allowed-tools` or
    a Claude Code upgrade.

## M4: xhigh and max tier

Setup for items 41 to 54: a throwaway repo with a migration file, two areas of code that
share no file, and two open issues (#1, the common one-line bug, and #2, a second change
in the other area), and Codex installed unless the item says otherwise. An item that
names an Opus-qualifying slice describes it in its setup, since the common one-line bug
never qualifies. Where an expectation names the model of an Agent or Workflow call, read
it from the session's tool trace, not from the plan or the report.

41. **Max tier reviews use `gpt-6-astra`.** Command: `/ccl:run #1 --effort max`.
    Expected: the report records a `gpt-6-astra` thread for Step 3 and another for Step
    5, a `code-review xhigh` pass per Step 5 round, and no Codex thread for Step 4. Rerun
    after any change to the roles table.
42. **A requested xhigh tier stands above the floor.** Command:
    `/ccl:run "add a column" --effort xhigh`. Expected: the run is xhigh, the report
    says the floor was applied and that the requested tier was above it, and both Step 3
    and Step 5 use `gpt-6-astra` because the trigger exists, with a `code-review high`
    pass at Step 5. Rerun after any change to the risk floor or the trigger rule.
43. **Xhigh with a single-slice plan.** Command: `/ccl:run #1 --effort xhigh` for a
    one-line bug. Expected: the plan has one slice, Step 4 makes one Agent call, and the
    implementer is Sonnet, with the reason that no Opus criterion applies. Rerun after
    any change to Step 4.2 or to the Implementer choice section of `tiers.md`.
44. **Re-evaluation does not raise an xhigh run.** Setup: a task sized xhigh whose
    implementation ends up removing an auth check. Command: `/ccl:run #1 #2`. Expected:
    the tier stays xhigh, the report says the floor applied at re-evaluation and the tier
    was unchanged, Step 3 used `gpt-6-astra` (xhigh always does), and Step 5 runs with
    `gpt-6-astra` because the diff has a trigger, beside a `code-review high` pass. Rerun
    after any change to the re-evaluation rule or the trigger rule.
45. **Parallel slices at high tier.** Setup: one cross-cutting change in one deliverable
    whose two parts touch disjoint files, each part about one subagent timeout of work.
    Command: `/ccl:run #1 --effort high`. Expected: one Workflow, or parallel Agent calls
    in one message, with two Sonnet agents and the choice logged in `run.md`, the tier
    still high, and a run log showing two round counters, one per slice. Rerun
    after any change to the Step 2 slice rule, Step 4.2, or the Budgets section.
46. **Ordered slices at medium tier.** Setup: several files in one area, about two
    subagent timeouts of work in total, where the first slice creates a helper in a file
    it owns and the second slice, in files only it owns, calls that helper. Command:
    `/ccl:run #1 #2 --effort medium`. Expected: two Agent calls in order, the second not
    started before the first ends, both Sonnet, the tier medium, and no file in both
    slices. Rerun after any change to the Step 2 slice rule or Step 4.2.
47. **Opus by the contract criterion at xhigh.** Setup: #1 asks for a new module in one
    area and #2 asks the other area to call it, so the plan has two slices and the first
    adds a module the second cites. Command: `/ccl:run #1 #2 --effort xhigh`. Expected:
    `opus` for the first slice in the plan and in the report, with the contract criterion
    named, and the Agent or Workflow call for that slice made with model `opus` in the
    tool trace. Rerun after any change to the Implementer choice section of `tiers.md`.
48. **Opus by the risk criterion at max.** Setup: #1 asks for an auth check in one file.
    Command: `/ccl:run #1 --effort max`. Expected: `opus` for that slice in the plan and
    in the report, with the risk criterion named, and the Agent call made with model
    `opus` in the tool trace. Rerun after any change to the Implementer choice section of
    `tiers.md` or to the risk floor.
49. **Opus tool error falls back to Sonnet.** Setup: the item 48 issue, so the slice
    qualifies for `opus` by the risk criterion, and a session in which an Agent call at
    model `opus` returns an error, for example a session whose model set has no `opus`.
    Command: `/ccl:run #1 --effort xhigh`. Expected: the tool trace shows the `opus` call
    erroring and the same prompt sent at `sonnet`, `run.md` has the error, the report
    names the implementer swap, and a Step 5 fix for that slice is also sent at `sonnet`.
    Rerun after any change to the Opus fallback rule.
50. **Sonnet chosen for a small max slice.** Setup: a max run whose second slice is one
    documentation file with no risk trigger and no new section that anything cites.
    Command: `/ccl:run #1 #2 --effort max`. Expected: `sonnet` for that slice, with the
    reason "none" in the plan and the report. Rerun after any change to the Implementer
    choice section of `tiers.md`.
51. **Denial during an Opus call.** Setup: the item 48 issue, so the slice qualifies for
    `opus`, and default permission mode; deny the implementer's first write when it
    prompts. Command: `/ccl:run #1 --effort xhigh`. Expected: the tool trace shows the
    implementer at `opus` and no later call at `sonnet`, the denial handled by Approval
    scope carve-out 3, and the run ends `blocked`. Rerun after any change to Approval
    scope carve-out 3 or the Opus fallback rule.
52. **The file threshold.** Setup: #1 asks for the same one-line edit in exactly eight
    named files in one area, and #2 for the same edit in nine named files in the other
    area, with no risk trigger and no new cited section in either. Command:
    `/ccl:run #1 #2 --effort xhigh`. Expected: the eight-file slice on `sonnet` and the
    nine-file slice on `opus` with the file criterion named, in the plan, in the report,
    and in the models of the two calls in the tool trace. Rerun after any change to the
    Implementer choice section of `tiers.md`.
53. **A timeout is not a fallback.** Setup: the item 48 issue, so the slice qualifies for
    `opus`, and `.ccl.json` set to `{"timeouts": {"subagent": 1}}` so the call runs past
    its budget. Command: `/ccl:run #1 --effort xhigh`. Expected: the tool trace shows the
    implementer at `opus` and no later call at `sonnet`, and the run ends `blocked`
    naming the subagent budget. Rerun after any change to the Budgets section or the Opus
    fallback rule.
54. **Effective model after a Workflow.** Setup: #1 is the item 48 auth check in one
    area, and #2 is a one-line change in the other area that shares no file with it, so
    the plan has two independent slices and the first qualifies for `opus` by the risk
    criterion. The session makes an `opus` call error as in item 49, and the first
    slice's diff must draw a blocking finding so a Step 4.3 round needs a fresh agent.
    Command: `/ccl:run #1 #2 --effort xhigh`. Expected: with one Workflow, the tool trace
    shows one Workflow with both slices, its `opus` call erroring, the rerun at `sonnet`,
    and the fresh agent at `sonnet`, and the report names one swap. With parallel Agent
    calls in one message, the choice is logged in `run.md`, the `opus` call errors and is
    rerun at `sonnet`, and a continued agent, not a fresh one, is acceptable for the
    review round, still at `sonnet`, with one swap in the report. Rerun after any change
    to Step 4.2 or the Opus fallback rule.

## M5: the second final reviewer

Setup for items 55 to 61: the M3 setup, with the built-in `code-review` skill listed in
the session unless the item says otherwise.

55. **High tier without a trigger uses `gpt-6.1-sol` and `code-review medium`.** Setup: a
    cross-cutting change inside one deliverable that touches no risk floor area. Command:
    `/ccl:run #1 --effort high`. Expected: `gpt-6.1-sol` threads for Step 3 and Step 5, one
    `code-review medium` call per Step 5 round in the tool trace with the level passed
    explicitly and neither `--comment` nor `--fix`, the report's Claude review passes line
    filled per round, and the run log showing both passes returned before any Step 5 fix.
    Rerun after any change to Step 5, the Claude review contract, or the trigger rule.
56. **Floored high tier uses `gpt-6-astra` for the plan review only.** Command:
    `/ccl:run "add a column" --effort high`. Expected: a `gpt-6-astra` thread for Step 3,
    a `gpt-6.1-sol` thread for Step 5, and a `code-review medium` pass. Rerun after any
    change to the trigger rule.
57. **A missing `code-review` skill blocks only where it is needed.** Setup: a session in
    which the `code-review` skill is not listed. Each run gets its own branch name, since
    the first run's default branch would otherwise exist and block the last run at Step
    3.7.2 before it reaches Step 5. Command: `/ccl:run #1 --effort medium --branch t57-a`,
    then `/ccl:plan #1 --effort high`, then `/ccl:run #1 --effort high --branch t57-b`.
    Expected: the first ends `done` with no mention of the skill, the second ends
    `plan-only` with no mention of the skill, and the third ends `blocked` at the start of
    Step 5, after implementation, naming the missing skill. Rerun after any change to the
    Claude review contract.
58. **The Claude pass reviews the base-to-working-tree diff and nothing else.** Setup: a
    clone whose local `main` is two commits behind the remote default branch, so the
    skill's own range would include commits the task did not make; the item 14 shape, but
    a CI failure the loop can fix in one edit, at high tier. Command:
    `/ccl:run #1 --effort high`. Expected: every `code-review` call in the tool trace
    carries the level (`medium` at high tier) and `<base-sha>...HEAD` with the full base
    SHA; no finding, before the push or in the repair cycle, names a file only the two
    stale commits touched; and the repair cycle's pass covers the committed task files and
    the uncommitted repair. If the target is ignored or rejected, record it, stop, and
    revisit the Claude review contract item 3. Rerun after 0.6.0 is installed, after any
    Claude Code upgrade, and after any change to Step 5.2 or Step 7.3.5. The 2026-09-29
    run showed that a bare commit target is not honored. The range target was verified
    directly on 2026-09-30 (see the record of runs); this item confirms it end to end
    after 0.6.0 is installed and after any Claude Code upgrade.
59. **A blocker found by Claude alone gates publication.** Setup: a slice that plants one
    defect a diff review should catch, such as a dropped error return, at high tier.
    Command: `/ccl:run #1 --effort high`. Expected: the run log's merged findings list
    names the source of each finding; the defect is fixed by the slice's implementer before
    any push; and the next round shows both a Codex follow-up in the Step 5 thread and a
    fresh `code-review medium` call over the fixed diff, with neither pass started before
    the fix batch ended. The item proves its point only when the log records the defect
    with Claude as its only source; if Codex reported it too, the item is inconclusive and
    is rerun with a different planted defect. Rerun after any change to Step 5.3 or 5.4.
60. **A Step 5 fix is not published unreviewed.** Setup: as item 59. Expected: for every
    edit `run.md` records during Step 5, a later round lists both passes over a diff that
    includes it, or the run did not reach Step 7. Rerun after any change to Step 5.4 or the
    Budgets section.
61. **The third round fixes nothing.** Setup: a run that reaches a third Step 5 round;
    record how, for example a slice whose fix in round two draws a new confirmed finding.
    Expected: if the third round has a confirmed blocking finding, the run ends `blocked`
    with the finding in the report, no edit after the third round in `run.md`, no
    orchestrator fix, and nothing pushed; if it has only non-blocking findings, they are
    listed as deferred with the reason that no round remained, `run.md` shows no edit after
    the third round, and the run continues to Step 6. Rerun after any change to Step 5.3,
    5.4, or the Budgets section.

## M6: Live-run amendments of 2026-09-29

Setup for items 62 to 82: the common setup, plus the setup each item names. Items 62 to
82 are hand runs against throwaway repos and cannot run inside a ccl run.

62. **Non-GitHub remote.** Setup: a throwaway repo whose `origin` URL host is not
    `github.com` (a local bare repository or an Azure DevOps URL) and no `gh` login for
    that host, so `gh repo view` fails.
    Command: `/ccl:run #1 --no-codex`, then `/ccl:run "rename the README heading"
    --no-codex`. Expected: the first is rejected by the command with "issue inputs are
    not accepted on a non-GitHub host; pass a file or a description" and no file is
    written; the second runs Steps 0 to 6, never runs Step 7, and ends `prepared`, and
    the report names the host and says publication is handed to the repo's own tooling.
    Rerun after any change to Step 0 host detection, Step 7, or the commands' Step 2.
63. **`--no-publish` ends `prepared`.** Command: `/ccl:run #1 --no-codex --no-publish`.
    Expected: `prepared`, no push and no PR, the work uncommitted on the local branch, and
    a report that gives the `git add` and `git commit` commands, the `git push -u <remote>
    <branch>` command, and the `gh pr create` command. Rerun after any change to Step 7
    or the terminal states.
64. **A refused Step 7 prompt.** Setup: a user instruction file with an ask-first rule
    for push. Command: `/ccl:run #1 --no-codex`, answering "no" to the push prompt.
    Expected: `prepared`, nothing pushed, and the report naming the branch and the
    publish commands. Then repeat with an ask-first rule for the PR only, answering "yes"
    to the push and "no" to the PR: the branch is pushed and the run ends `blocked`, since
    a denial after a push is not withheld publication. Rerun after any change to Approval
    scope carve-out 3 or Step 7.1 and 7.2.
65. **`--no-publish` with `"commit": true`.** Setup: `.ccl.json` with `{"commit": true}`,
    committed. Command: `/ccl:run #1 --no-codex --no-publish`, then `git status` and
    `git log`. Expected: `prepared`, nothing under `specs/ccl/`, the tree uncommitted, no
    new commit, and the report giving the commit commands. Rerun after any change to Step
    7.1 or the `commit` field.
66. **Codex availability without the skill list.** Setup: codex-lite 0.7.0 or later
    installed and enabled, `codex` on PATH, and a session whose skill list omits
    `codex-lite:ask`. Command: `/ccl:run #1`. Expected: Codex is treated as available,
    `run.md` records the check as `codex --version` and the plugin version, and the Step
    3 call is attempted. If the Skill call errors because the skill is not listed, the
    call counts as `failed`: it is retried once with the same arguments, then the stage
    swaps to the Claude fallback, and the report names the swap with the reason "skill
    not listed in session". Rerun after any change to Codex availability or the Reviewer
    contract.
67. **Line endings.** Setup: a repo whose tracked files use CRLF, and an issue that asks
    for one new file. Command: `/ccl:run #1 --no-codex`. Expected: the new file has the
    ending the implementer rule defines, the `eol=` attribute first, else no comparison
    under `text` or `text=auto`, else the majority in its directory, else the majority in
    the repository, here CRLF, read from the `w/` column of `git ls-files --eol`
    (`w/crlf`), no edited file changed its endings, and a new
    binary file (`w/-text`) or a new file with no line ending (`w/none`) draws no finding.
    Then plant an LF new file in a slice and rerun: Step 4.3 raises a
    finding that names the file, and the implementer fixes it before review. Rerun
    after any change to the Implementer prompt or Step 4.3.
68. **Parallel Agent calls.** Setup: two independent slices in two areas that share no
    file, at medium tier, with the first slice's diff drawing a blocking finding. Command:
    `/ccl:run #1 #2 --effort medium`. Expected: both implementers start in one message as
    parallel Agent calls, the choice and the reason are logged in `run.md`, and the
    review round goes to the same agent through SendMessage. Rerun after any change to
    Step 4.2 or 4.3.
69. **Run budget flag and tier default.** Command: `/ccl:run #1 --no-codex --run-budget
    1`, then `/ccl:run #1 --no-codex --effort medium`. Expected: the first ends `blocked`
    naming the run budget, with the flag as its source in `run.md` and the report; the
    second reports a budget of 120 minutes with the source "tier default". Rerun after
    any change to the Budgets section.
70. **Session instruction changes the run budget.** Setup: `.ccl.json` with
    `{"timeouts": {"run": 1}}`. Command: `/ccl:run #1 --no-codex`, and while it runs send
    a message naming a new budget, for example "extend the run budget to 60 minutes".
    Expected: `run.md` records the new budget and the time it took effect, the run
    continues, and the report names 60 minutes with "session instruction" as the source.
    Rerun after any change to the Budgets section.
71. **Dropped calls.** Setup: a session in which the permission mode drops a read-only
    call and a write call without denying either, for example by a classifier that
    returns nothing. Command: `/ccl:run #1 --no-codex`. Expected: the dropped read-only
    call is retried once, serially, and recorded in `run.md`, the run is not `blocked`,
    and a dropped write whose target check shows it took effect is recorded as done and
    not repeated. An explicit denial in the same session still ends `blocked`. Rerun
    after any change to Approval scope carve-out 6 or Step 0.
72. **Skip-worktree files at low tier.** Setup: after `git update-index --skip-worktree
    <file>` and an edit to that file, `git status --porcelain` prints nothing and `git
    ls-files -v` shows `S` for it. Command: `/ccl:run #1 --no-codex --effort low`.
    Expected: the run sees the clean status and the flagged file that differs from
    `HEAD`, creates a detached worktree beside the checkout, at
    `<checkout-parent>/<checkout-name>-ccl-<run-id>`, with `git worktree add --detach`,
    records it in `run.md`, works in it, skips Step 5 at low tier, and the
    report names the worktree path and `git worktree remove <path>`. The Step 0.1
    statement, printed before any worktree exists, already lists the `git worktree add`
    and `cd <checkout> && ...` prompts, and the report says the run was attended. Rerun
    after any change to Step 0.1, Step 0.3, or `worktree.md`.
73. **Skip-worktree files at medium tier.** Setup: as item 72, plus a `test` script that
    fails on the original tree's skip-worktree state and passes at the base commit, and
    codex-lite installed, and a lockfile and an install step the instruction files name
    (for example `npm ci`). Command: `/ccl:run #1 --effort medium`. Expected: the install
    step runs in the worktree before the baseline, the Step 5
    diff review goes through `codex-lite:ask` with a patch file and no `codex-lite:review`
    call appears in the tool trace; the baseline and Step 6 checks run in the worktree,
    which the passing `test` shows; and the PR's head branch equals the branch the run
    created in the worktree. Rerun after any change to Step 0.3, Step 5.2, `worktree.md`,
    or the Reviewer contract.
74. **Skip-worktree files at high tier.** Setup: as item 72. Command: `/ccl:run #1
    --effort high`. Expected: `blocked` at Step 1.5 naming the skip-worktree files that
    differ from `HEAD` and the tier, nothing implemented, and the report written to
    the run directory. Rerun after any change to Step 0.3, Step 1.5, or `worktree.md`.
75. **Two-repo run at medium tier.** Setup: two throwaway GitHub repos on the same host,
    each with one open issue (#1 in each) that describes a one-line bug, the second
    checked out beside the first, and codex-lite installed. Command: run from the primary
    `/ccl:run #1 <URL of the second repo's issue> --repo <path of the second repo>`.
    Expected: one branch with the same name in each repo, one PR per repo with the sibling
    links filled by `gh pr edit`, `Closes` only from the issue's own repo and `Refs
    <owner>/<repo>#n` from the other, one comment per issue naming both PRs with the
    issue's own repo first, a Codex review of the primary by `codex-lite:review`, and of
    the second repo through `codex-lite:ask` with `diff-<slug>.patch`, the second repo's
    PR opened against the second repo, its CI read from the second repo, and each PR
    body in `.ccl/<run-id>/pr-body-<slug>.md` in the primary, with every `--body-file` of
    a `gh` call for the second repo an absolute path. Rerun after any change to
    `multi-repo.md` or Step 7.
76. **Multi-repo host and bare `#n` rules.** Setup: as item 75, plus a third checkout whose
    `origin` is on a different host, and an issue number that exists only in the
    second repo. Command: `/ccl:run "x" --repo <third checkout>`, then `/ccl:run #<n>
    --repo <second repo>` for the number that exists only there. Expected: the first is
    rejected before Step 0.1 naming both hosts; the second is rejected by the command
    because a bare `#n` is checked against the primary only. Nothing is written in
    either case. Rerun after any change to `multi-repo.md` or the commands'
    Step 2.
77. **Two-repo run at high tier.** Setup: as item 75, each repo changed. Command: as
    item 75 with `--effort high`. Expected: one `code-review medium` call over the
    primary's diff, and one Opus Agent call for the second repo's
    Claude slot, both recorded in the report's Claude review passes line, and the Opus
    call named as a substitute, not a swap. Rerun after any change to the Claude review
    contract or `multi-repo.md`.
78. **Skip-worktree primary in Multi-repo mode.** Setup: as item 72 for the primary, plus
    a second repo. Command: `/ccl:run #1 --repo <second repo>`. Expected: `blocked` at
    Step 0.3 naming the skip-worktree files that differ from `HEAD` and saying the
    worktree exception does not apply in Multi-repo mode, and nothing written. Rerun
    after any change to Step 0.3 or `multi-repo.md`.
79. **A worktree run raised to high at Step 4.5.** Setup: as item 72, at medium tier, with
    an issue whose implementation removes an auth check. Command: `/ccl:run #1 --effort
    medium`. Expected: Step 4.5 raises the tier to high and the run ends `blocked` there
    naming the skip-worktree files that differ from `HEAD` and the tier, with no Step 5
    call in the tool trace and the
    report naming the worktree path. Rerun after any change to Step 4.5, Step 0.3, or
    `worktree.md`.
80. **A denied push with `"commit": true`.** Setup: `.ccl.json` with `{"commit": true}`,
    committed, and an ask-first rule for push. Command: `/ccl:run #1 --no-codex`,
    answering "no" to the push prompt. Expected: `prepared`, the branch carries one commit
    that holds `specs/ccl/<run-id>/plan.md` and `specs/ccl/<run-id>/report.md` in state
    `publishing`, and the report says so and gives the push and `gh pr create` commands.
    Rerun after any change to Step 7.1 or the `commit` field.
81. **Only the additional repo changes.** Setup: as item 75, with an issue that changes only
    the second repo. Command: as item 75 at medium tier, with `commit` false (the common
    setup); with `commit` true, the snapshot goes to the first repository with a diff,
    here the second repo. Expected: no `codex-lite:review`
    call in the tool trace, one fresh `codex-lite:ask` thread naming `diff-<slug>.patch`
    recorded as the Step 5 thread, `run.md` naming the primary as skipped with an empty
    diff, and one PR, in the second repo, and none in the primary. Rerun after any change
    to Step 5.2 or `multi-repo.md`.
82. **Multi-repo with `--no-codex`.** Setup: as item 75, both repos changed, with a
    planted defect in the second repo's slice. Command: as item 75 with `--no-codex` at
    medium tier. Expected: one Opus Agent call for Step 5 given both `diff.patch` and
    `diff-<slug>.patch`; the defect fixed and the next round sent to the same agent with
    SendMessage; a CI repair in either repo reviewed by that same agent; and the report
    naming the swap once and both patch files. Rerun after any change to Step 5.2, Step
    7.3.5, or `multi-repo.md`.

## M7: 0.6.0, 2026-09-30

Setup for items 83 to 124: the common setup, plus the setup each item names. Items 83 to
99 and 101 to 124 are hand runs against throwaway repos and cannot run inside a ccl run.
Item 100 is a static check of the plugin files and needs no repo. In an item that
continues a branch, "the branch" is already pushed to the remote with one commit, "the
remote head" is that branch's head on the remote, and the session's checkout is on the
branch or detached at the remote head unless the item says otherwise.

83. **An open PR is continued.** Setup: branch `t83` pushed, with an open PR #2 whose base
    branch is `release`, not the default branch, and a pull request workflow with `on:
    pull_request: types: [synchronize]` and `branches: [release]`. Local `t83` absent or
    equal to the remote, and the session detached at `<remote>/t83`. Command: `/ccl:run
    #1 --no-codex --continue t83`. Expected: Step 0.2 records the remote head as the base
    commit and PR #2 with its base branch `release`; `run.md` records `HEAD` as the
    planning snapshot, equal to the base commit; the run switches to `t83` without
    creating it; the push is `git push <remote> t83`, with no `-u` and no force flag,
    after a `git ls-remote --heads <remote> refs/heads/t83` compare with the base commit;
    every `git ls-remote` in the trace names `refs/heads/<branch>`; the tool trace has no
    `gh pr create` and no `gh pr edit` of PR #2; one `gh pr comment 2 --body-file
    <absolute path>` carries the continued-PR body, which has an "Issues" section with
    `#1: complete` or `#1: partial`, no `Closes` or `Refs` line, and the closing sentence
    about the PR's own body; the first CI watch counts the `synchronize` workflow as
    applying and reads required checks from `release`; and the report's Continued line
    names `t83` and PR #2.
    Then repeat with `--no-publish`: the run ends `prepared`, and the report gives `git
    push <remote> t83` and `gh pr comment 2 --body-file <absolute path>`, not `gh pr
    create`; the body file exists at that path and holds the continued-PR body. Rerun
    after any change to Step 0.2, Step 7, `pr-body.md`, or `ci-watch.md`.
84. **A branch with no PR gets one.** Setup: branch `t84` pushed, with no PR. Command:
    `/ccl:run #1 --no-codex --continue t84`. Expected: the run switches to `t84`, pushes
    with `git push <remote> t84` and no force flag, and opens the PR with `gh pr create
    --head t84` and the standard body, against the default branch; the tool trace has no
    `gh pr comment` on a continued PR; the first CI watch uses `opened`; and the report's
    Continued line says "no PR". Rerun after any change to Step 0.2 or Step 7.2.
85. **Only closed or merged PRs is a preflight failure.** Setup: branch `t85` pushed,
    with one closed PR and one merged PR and no open one. Command: `/ccl:run #1
    --no-codex --continue t85`. Expected: a preflight failure in Step 0.2 naming both
    PRs, the report printed, nothing written, no branch switched, and nothing pushed.
    Rerun after any change to Step 0.2.
86. **A local branch that differs from the remote is a preflight failure.** Setup: branch
    `t86` pushed, and a local `t86` with one more commit that is not on the remote.
    Command: `/ccl:run #1 --no-codex --continue t86`. Expected: a preflight failure
    saying the local branch does not equal `<remote>/t86`, nothing written, and the
    local `t86` at the same commit as before, because the run never resets local work.
    Rerun after any change to Step 0.2.
87. **A remote that moved before the push ends `blocked`.** Setup: branch `t87` pushed,
    a second clone that can push to it, and `.ccl.json` with a `checks` entry that runs
    `sleep 90`, committed. Command: `/ccl:run #1 --no-codex --continue t87`, and while
    the check runs push one commit to `t87` from the second clone. Expected: `blocked`
    before the first push, naming `t87`, after a `git ls-remote --heads <remote>
    refs/heads/t87` compare; the remote head is the second clone's commit;
    the tool trace has no push by the run, no force flag, no `git rebase`, and no `git
    reset`; and the local branch is kept. Then repeat with `--confirm-plan`, pushing
    during the wait and answering yes: `blocked` naming `t87` before Step 3.7.2, with
    the branch not switched. Rerun after any change to Step 3.5 or Step 7.2.
88. **`--continue` with `--branch` is rejected by the command.** Command: `/ccl:run #1
    --continue t83 --branch x`, and `/ccl:plan #1 --continue t83 --branch x`. Expected:
    both rejected before Step 0 with a one-line message naming the two flags, the skill
    not loaded, and no file written. Rerun after any change to the commands' flag rules.
89. **`--continue` on a non-GitHub host ends `prepared`.** Setup: as item 62, plus a
    branch `t89` on the bare remote. Command: `/ccl:run "rename the README heading"
    --no-codex --continue t89`. Expected: the command's `git ls-remote --heads <remote>
    refs/heads/t89` check passes; Step 0.2 fetches `t89` and records `<remote>/t89` as the
    base commit; the tool trace has no `gh pr list`; the run switches to `t89`, runs Steps
    0 to 6, never runs Step 7, and ends `prepared`; and the report gives `git push
    <remote> t89`, names the host, and has no `gh pr create` command. Rerun after any
    change to Step 0 host detection or Step 0.2.
90. **Multi-repo with the branch on the primary only.** Setup: as item 75, with branch
    `t90` pushed to the primary's remote with an open PR, and no `t90` on the second
    repo's remote. Command: as item 75 with `--continue t90`. Expected: the primary
    switches to `t90` and the second repo creates `t90` from its default branch; the
    primary has no `gh pr create` and no body edit, and the second repo gets a new PR
    whose body says "pending" for its sibling link; the comment on the continued PR is
    posted only after the second repo's PR is open, with its "Related pull requests"
    section filled; `gh pr edit` covers only the second repo's PR; and each issue comment
    names both PRs. Rerun after any change to `multi-repo.md` or Step 7.2.
91. **An unnamed writable checkout ends `blocked` in Step 1.** Setup: an issue whose text
    asks for a change in a second checkout on disk, for example "also update
    `../other/README.md`", and a run with no `--repo`. Command: `/ccl:run #1
    --no-codex`. Expected: `blocked` in Step 1 before Step 2, nothing implemented, and a
    report with the rerun command: the same inputs and flags plus `--repo <path>`, with
    the path from the issue. The run does not adopt Multi-repo mode. Then change the
    issue so the second checkout is only read: the run is not blocked there. Then ask for
    changes in two other checkouts and pass `--repo` for one of them: `blocked` in Step
    1 again, and the rerun command adds one `--repo` for the missing checkout only. Rerun
    after any change to Step 1.4.
92. **An additional repo with a differing skip-worktree file continues.** Setup: as item
    75, plus `git update-index --skip-worktree <file>` in the second repo and an edit to
    that file, so `git status --porcelain` is empty there and the file differs from
    `HEAD`. Command: as item 75 at medium tier. Expected: the run is not `blocked` at
    Step 0.3; `run.md` records the file under the second repo; the report names it as
    local state the second repo's baseline and Step 6 checks ran against; and the file is
    absent from `diff-<slug>.patch`. Rerun after any change to Step 0.3 or
    `multi-repo.md`.
93. **Times appear only at the named points, in one form.** Command:
    `/ccl:run #1 --no-codex --effort medium`, then read `run.md` and the tool trace.
    Expected: every time in `run.md` is in the form `2026-09-30T14:05:09Z`, and each one
    equals the output of a `date -u +%Y-%m-%dT%H:%M:%SZ` call made just before the entry;
    times appear only at the start of Step 0, at the start of each step that has a
    `## Step` heading, before and after each timed call (Agent, Workflow, SendMessage,
    Codex, `code-review`, each check, each install step, each CI poll), and at the
    terminal state; no time is logged for an untimed `git` call or a substep; and each
    elapsed figure equals the
    difference between the Step 0 start and a recorded time, less each Step 3.5 wait.
    Rerun after any change to the Budgets section.
94. **`--confirm-plan` answered yes continues.** Command: `/ccl:run #1 --no-codex
    --confirm-plan`; when the plan is printed, wait about 3 minutes and reply "yes".
    Expected: the run prints the plan path and a short summary, asks, and ends its turn;
    after the reply it goes on through Step 3.6 and Step 3.7 to a normal terminal state;
    `run.md` holds the question time and the reply time; the report's plan approval line
    gives the question, the reply, and the wait; and the elapsed time against the run
    budget is the Step 0 start to the latest recorded time, minus that wait. In default
    permission mode the Step 0.1 statement lists the plan approval question as a prompt.
    Rerun after any change to Step 3.5, the Budgets section, or `report.md`.
95. **`--confirm-plan` answered with a change.** Command: as item 94, replying with a
    change, for example "also add a test for the empty input". Expected: the change is
    recorded in `inputs.md` as an ad-hoc input; `run.md` shows one more Step 3 round,
    within the cap of 3; the plan is revised; the question is asked again with a new pair
    of times; and a "yes" to the second question continues the run. Rerun after any
    change to Step 3.5.
96. **`--confirm-plan` answered no.** Command: as item 94, replying "no", and again
    replying "not now". Expected: each run ends `plan-only`, with `plan.md` written, no
    branch, no check run, no push, and the report's plan approval line quoting the reply.
    Rerun after any change to Step 3.5 or the terminal states.
97. **A change when no Step 3 round is left ends `stopped`.** Setup: a task whose Step 3
    uses all 3 rounds, for example one whose plan draws a blocking objection in each of
    the first two rounds and a revision in the third. Command: as item 94, replying with
    a change. Expected: `stopped`, with the requested change as the question in the
    report and the statement that no plan review round was left, no branch created, and
    nothing implemented. A rerun with the change as an extra ad-hoc input starts from
    Step 0. Rerun after any change to Step 3.5 or the round cap.
98. **A planning snapshot behind the base is revised before the question.** Setup: as item
    10, plus a change on the default branch to a file the plan cites, so the plan must
    change. Command: `/ccl:run #1 --no-codex --confirm-plan`. Expected: `run.md` logs the
    reverification against the base commit, and the revision with its Step 3 round, before
    the question is asked; the plan the user sees already matches the base commit; and
    after the reply Step 3.7 does not repeat the reverification. Rerun after any change
    to Step 3.5 or Step 3.7.1.
99. **`--confirm-plan` is rejected with `--plan-only` and on `/ccl:plan`.** Command:
    `/ccl:run #1 --plan-only --confirm-plan`, then `/ccl:plan #1 --confirm-plan`.
    Expected: the first is rejected as a conflict of the two flags, and the second as not
    applicable, with a pointer to `/ccl:run --confirm-plan`; in both the skill is not
    loaded and no file is written. Rerun after any change to the commands' flag rules.
100. **Moved text is unchanged and the default read is 1,115 lines.** This is a static
     check of the plugin files, not a run, and it needs no repo. Command: take every line
     that `git diff 0ba8c61^ 0ba8c61 -- skills/ccl/SKILL.md` removes and compare each,
     with leading spaces trimmed, against the lines of `skills/ccl/worktree.md`,
     `skills/ccl/multi-repo.md`, and `skills/ccl/ci-watch.md` as of 0ba8c61 (`git show
     0ba8c61:<path>`); then run `wc -l` on `skills/ccl/SKILL.md` and `skills/ccl/tiers.md`
     as of 0ba8c61. Later rule commits change the files, so the counts hold at 0ba8c61
     only. Expected: every removed line that
     holds a rule is in one of the three files as a whole line; the only removed lines
     not found are the 11 lines `SKILL.md` rewrote in place for the pointers and
     cross-references, and every rule sentence in them is still in `SKILL.md` word for
     word apart from the reference to the moved section; and `SKILL.md` has 934 lines and
     `tiers.md` 181, so a default single-repository GitHub run reads 1,115 lines through
     Step 6. Rerun after any edit that moves text between these files.
101. **A `--continue` value that is not a branch name is rejected by the command.**
     Command: `/ccl:run #1 --continue -x`, `/ccl:run #1 --continue 'a@{1}'`, `/ccl:run #1
     --continue 'a..b'`, `/ccl:run #1 --continue 'a;b'`, `/ccl:run #1 --continue 'a$b'`,
     and `/ccl:plan #1 --continue 'a..b'`. Expected: each is rejected
     before Step 0 with a one-line message naming the value, the skill not loaded, and no
     file written. Then `/ccl:run #1 --no-codex --continue t83` with a valid pushed
     branch is not rejected for its name. Rerun after any change to the commands' flag
     rules.
102. **A branch checked out in another worktree is a preflight failure.** Setup: branch
     `t102` pushed, local `t102` equal to the remote, and `git worktree add <path>
     t102` so another checkout holds it. Command: `/ccl:run #1 --no-codex --continue
     t102` from the main checkout. Expected: a preflight failure in Step 0.2 naming that
     worktree path, the report printed, nothing written, no branch switched, and nothing
     pushed. Rerun after any change to Step 0.2 or `worktree.md`.
103. **A dirty tree after the Step 3.5 wait ends `blocked`.** Command: `/ccl:run #1
     --no-codex --confirm-plan`; when the plan is printed, edit a tracked file, then
     reply "yes". Expected: `blocked` before Step 3.7.2 naming the changed file, no
     branch created, and nothing implemented. Then with a pushed branch `t103`, local
     `t103` equal to the remote, and `--continue t103`, commit to local `t103` during the
     wait: `blocked` naming the branch, with the branch not switched. Rerun after any
     change to Step 3.5.
104. **`--continue` with `HEAD` off the base commit is a preflight failure.** Setup:
     branch `t104` pushed, local `t104` absent or equal to the remote, and the session on
     the default branch. Command: `/ccl:run #1 --no-codex --continue t104`, then
     `/ccl:plan #1 --continue t104`. Expected: each is a preflight failure in Step 0.2
     whose message gives `git switch t104` or `git switch --detach <remote>/t104`, the
     report printed, nothing written, no branch switched, and nothing pushed. Then run
     `git switch --detach <remote>/t104` and rerun the first command: it is not refused
     for `HEAD`, `run.md` records `HEAD` as the planning snapshot, and the plan review
     and the Step 1.2 reproduction read that code. Then add a `skip-worktree` edit so a
     worktree run is needed: with the session on `t104` the run ends `blocked` naming the
     session's checkout, and with the session detached at `<remote>/t104` it does not.
     Rerun after any change to Step 0.2 or `worktree.md`.
105. **A plan-only `--continue` records the conditions that matter only for building.**
     Setup: `HEAD` at the base commit, and in turn one of these for branch `t105`: a
     local `t105` with one commit that is not on the remote; `t105` checked out in
     another worktree; only closed or merged PRs; two open PRs. Command: `/ccl:plan #1
     --no-codex --continue t105`, and `/ccl:run #1 --no-codex --plan-only --continue
     t105`. Expected: no preflight failure in any case; each run ends `plan-only` with
     `plan.md` written, and `run.md` and the report record the condition. The same
     setups with `/ccl:run #1 --no-codex --continue t105` still fail preflight, as in
     items 85, 86, and 102. Rerun after any change to Step 0.2.
106. **A plan that edits a flagged path in an additional repo ends `blocked`.** Setup: as
     item 92, with an issue whose fix needs a change to the skip-worktree file. Command:
     as item 92. Expected: `blocked` at Step 2 naming the path, and nothing implemented.
     Then with an issue that needs no change to that file, every implementer prompt for
     the second repo names the path as off limits. Rerun after any change to
     `multi-repo.md`.
107. **The Step 3.5 rechecks cover `HEAD` and the worktree.** Setup: branch `t107`
     pushed, local `t107` equal to the remote. Command: `/ccl:run #1 --no-codex
     --confirm-plan --continue t107` with the session on `t107`; when the plan is
     printed, run `git switch` to the default branch, then reply "yes". Expected:
     `blocked` before Step 3.7.2 naming `HEAD`, with the branch not switched. Then with
     the session detached at `<remote>/t107`, run `git worktree add <path> t107` during
     the wait and reply "yes": `blocked` naming that worktree. Rerun after any change to
     Step 3.5.
108. **A `prepared` `--continue` run in Multi-repo mode writes each body to its own
     path.** Setup: as item 90, with branch `t108` pushed to both remotes, each with an
     open PR. Command: as item 90 with `--no-publish`. Expected: `prepared`;
     `.ccl/<run-id>/pr-body.md` and `.ccl/<run-id>/pr-body-<slug>.md` both exist and
     hold the continued-PR body of their repository; and the report gives one `gh pr
     comment <n> --body-file <absolute path>` per repository, naming those files. Rerun
     after any change to Step 7 or `multi-repo.md`.
109. **`--continue` naming the default branch is a preflight failure.** Setup: none
     beyond the common setup. Command: `/ccl:run #1 --no-codex --continue main`, then
     `/ccl:plan #1 --continue main`, with `main` the default branch. Expected: each is a
     preflight failure in Step 0.2 naming the default branch, the report printed, nothing
     written, and nothing pushed. Rerun after any change to Step 0.2.
110. **The continued branch is fetched with an explicit refspec.** Setup: a single-branch
     clone of the default branch, with branch `t110` pushed to the remote. Command:
     `/ccl:run #1 --no-codex --plan-only --continue t110`. Expected: `git rev-parse
     <remote>/t110` succeeds after Step 0.2, and `run.md` records it as the base commit.
     Rerun after any change to Step 0.2.
111. **A plan-only `--continue` with a differing local branch needs a detached `HEAD`.**
     Setup: branch `t111` pushed, and a local `t111` with one commit that is not on the
     remote. Command: `/ccl:plan #1 --no-codex --continue t111` with the session on local
     `t111`. Expected: a preflight failure for `HEAD`, whose message gives `git switch
     --detach <remote>/t111`. Then run that command and rerun: the run ends `plan-only`
     and `run.md` and the report record the differing local branch. Rerun after any
     change to Step 0.2.
112. **A submodule of the primary is not another writable checkout.** Setup: the primary
     has a submodule at `sub/`, and an issue whose fix edits a file under `sub/`.
     Command: `/ccl:run #1 --no-codex --plan-only`. Expected: Step 1 does not end in
     `blocked` for a writable checkout not listed in `repos`. Then with an issue that
     edits a path in an unrelated git checkout: `blocked` with the rerun command. Rerun
     after any change to Step 1.4.
113. **A reverification revision with no round left ends `blocked`.** Setup: a plan whose
     Step 3 review used all 3 rounds and whose planning snapshot is not the base commit
     in a way that changes the plan; `--confirm-plan`. Command: `/ccl:run #1 --no-codex
     --confirm-plan`. Expected: `blocked` naming the unreviewed revision, with no
     question asked and no branch created. Repeat without `--confirm-plan` and expect the
     same at Step 3.7.1. Rerun after any change to Step 3.5 or Step 3.7.1.
114. **A worktree run posts the continued-PR comment with an absolute body path.** Setup:
     branch `t114` pushed with an open PR, a `skip-worktree` edit that differs from
     `HEAD`, and the session detached at `<remote>/t114`. Command: `/ccl:run #1
     --no-codex --continue t114`. Expected: the `gh pr comment` call runs inside the
     checkout with `--body-file` set to an absolute path under the original checkout's
     `.ccl/<run-id>/`, and the comment posts. Rerun after any change to Step 7.2 or
     `worktree.md`.
115. **A repository that lacks the continued branch creates it under that name.** Setup:
     Multi-repo mode with a second checkout whose remote lacks branch `t115`, and the
     primary's remote has it. Command: `/ccl:run #1 --no-codex --continue t115 --repo
     <path>`. Expected: both repositories end on a branch named `t115`, with no derived
     `feat/...` name, and the collision check runs for the second repository. Rerun after
     any change to Step 3.7.2 or `multi-repo.md`.
116. **A PR opened for a continued branch names the unreviewed commits.** Setup: branch
     `t116` pushed with two commits beyond the default branch and no open PR. Command:
     `/ccl:run #1 --no-codex --continue t116`. Expected: the PR opens against the default
     branch, and its body says under "Decisions for the reviewer" that the branch carries
     2 earlier commits this run did not review. Rerun after any change to Step 7.2 or
     `pr-body.md`.
117. **A `--no-publish` run records pull request states instead of failing.** Setup:
     branch `t117` pushed with two open PRs. Command: `/ccl:run #1 --no-codex --no-publish
     --continue t117`. Expected: preflight passes, `prepared` is reached, and the report
     names both PRs and gives no `gh pr comment` and no `gh pr create` command. Repeat
     with only a closed PR and expect the same pass and the same report. Repeat with a
     local `t117` that differs from the remote and expect a preflight failure. Rerun after
     any change to Step 0.2.
118. **The PR list covers more than 30 pull requests.** Setup: a branch with 31 closed
     PRs and one open PR, all from this repository. Command: `/ccl:run #1 --no-codex
     --continue t118`. Expected: the tool trace shows `gh pr list` with `--limit 100`, and
     the run finds the open PR. Rerun after any change to Step 0.2.

119. **Each push is timed.** Command: `/ccl:run #1 --no-codex`. Expected: `run.md` holds
     a `date` time taken right after the push returned, and the CI watch does not judge
     CI until 2 minutes after that time. Rerun after any change to the Budgets section or
     `ci-watch.md`.
120. **A changed PR state before the first push ends `blocked`.** Setup: branch `t120`
     pushed with one open PR. Command: `/ccl:run #1 --no-codex --confirm-plan
     --continue t120`; when the plan is printed, close the PR, then reply "yes".
     Expected: `gh pr list` runs again before the push, and the run ends `blocked` naming
     the closed PR, with nothing pushed. Repeat with no PR at the start and a PR opened
     during the wait, and again with a second PR opened, and expect `blocked` each time.
     Rerun after any change to Step 7.2.
121. **A requested change that leaves a blocking objection at the cap ends `blocked`.**
     Setup: a plan whose Step 3 review used 2 rounds; `--confirm-plan`. Command:
     `/ccl:run #1 --no-codex --confirm-plan`; reply with a change that draws a blocking
     objection the plan cannot resolve. Expected: the third round ends `blocked` naming
     the objection, and the question is not asked again. Rerun after any change to Step
     3.5.
122. **A same-owner fork with the same branch name is ignored.** Setup: branch `t122`
     pushed with one open PR, and a pull request from a fork with the same owner and a
     branch named `t122`. Command: `/ccl:run #1 --no-codex --continue t122`. Expected:
     the `gh pr list` call asks for `isCrossRepository`, the fork's PR is dropped, and the
     run continues with the one open PR. Rerun after any change to Step 0.2.
123. **A plan that needs an unlisted writable checkout ends `blocked` at Step 2.** Setup:
     an issue whose text does not name the other repository, and whose plan turns out to
     need edits in a second writable checkout not passed with `--repo`. Command:
     `/ccl:run #1 --no-codex`. Expected: `blocked` with the rerun command that adds
     `--repo <path-to-owner/repo>`, no branch created, and no `git submodule status` call
     in the tool trace. Rerun after any change to Step 1.4 or Step 2.
124. **The `HEAD` failure names the command that works.** Setup: branch `t124` pushed,
     and a local `t124` one commit behind the remote. Command: `/ccl:plan #1 --continue
     t124` with the session on local `t124`. Expected: a preflight failure whose message
     gives `git merge --ff-only <remote>/t124`. Then with the session on the default
     branch and local `t124` equal to the remote or absent: the message gives `git switch
     t124`. Then with local `t124` holding a commit that is not on the remote: the message
     gives `git switch --detach <remote>/t124`. Rerun after any change to Step 0.2.

## Record of runs

2026-09-29, item 58, partial: a `code-review medium <base-sha>` call reviewed the commit
itself and skipped the working tree; the run reran the pass without a target after
confirming local `main` equaled the base.

2026-09-30, direct test of the #15 range target, not an item run: a throwaway repo with
a local bare remote and a local `main` two commits behind it, one stale commit carrying a
planted bug. Headless `claude -p` called the `code-review` skill through the Skill tool,
as the orchestrator does (Claude Code 2.1.284). A bare commit target reviewed only that
commit and missed both task bugs. A pinned upstream was correct in one of two runs. The
target `<base>...HEAD` was correct in five of five runs, at low, medium, high, and xhigh,
in both the Step 5 state and the CI repair state.
