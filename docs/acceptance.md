# Acceptance checks

The plugin is prompt-only and has no automated test surface in 0.1.0, so these checks
are run by hand against a throwaway repo. Each item gives the setup, the command, the
expected result, and when to rerun it. The record of runs is at the end.

Common setup for items 4 to 141 unless an item says otherwise: a throwaway GitHub repo
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
   Opus subagent named as the `gpt-6.1-sol` swap, a Step 5 review by an Opus subagent
   named as the `gpt-6.1-sol` swap beside a `code-review low` pass, and an implementer at
   `sonnet`, named as the `--no-codex` fallback. Rerun after any change to Steps 0 to 7 or
   the templates.
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
codex-lite 0.8.0 or later installed and enabled, and `codex` on PATH, unless the item
says otherwise.

27. **Bundled issues at medium tier.** Command: `/ccl:run #1 #2`. Expected: `done`, at
    least one Step 3 round with the thread id recorded in `run.md`, a
    `codex-lite:implement` call at `gpt-6.1-sol` for the slice, one Step 5 review with a
    separate `gpt-6.1-sol` thread and one `code-review medium` call per Step 5 round in
    the tool trace, rejected
    findings listed with reasons, a PR body with a closing reference for each issue, and
    the tier medium, not high, because bundling alone does not raise it. Rerun after any
    change to Steps 3 to 5 or the tier rules.
28. **Parallel and ordered slices.** Setup: two areas of code that share no file, one
    plan with two independent slices, and one whose order of work makes the second
    depend on the first. Command: `/ccl:run #1 #2` for each. Expected: both runs are
    sized xhigh, because the change spans two areas that share no file; the slices of
    both runs are `codex-lite:implement` calls at `gpt-6-astra`, one after another, with
    no Workflow and no parallel Agent calls for them, and the dependent slices in order
    of the dependency. Then make the first slice of the independent run qualify for
    `opus` by the file criterion and rerun: its Agent call uses model `opus`, and the
    order of the two slices is logged in `run.md`. Rerun after any change to Step 4.2 or
    the estimate rule.
29. **No Codex.** Command: `/ccl:run #1 #2 --no-codex`. Expected: the same path with
    an Opus subagent for Step 3 and another for Step 5, the implementer at `sonnet`, the
    report names all three swaps, and one `code-review medium` call per Step 5 round
    appears in the tool trace, because `--no-codex` does not touch the Claude slot. No
    `codex-lite:implement` call appears. Rerun after any change to the fallback table.
30. **Codex missing from PATH.** Setup: codex-lite installed, `codex` removed from PATH.
    Command: `/ccl:run #1 #2`. Expected: the report names the swap and the reason for
    each reviewer stage, the implementer is `sonnet` by the Step 0.6 fallback, and no
    `codex-lite:implement` call appears. Rerun after any change to Step 0.6.
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
    high after Step 4, and the report shows a Step 5 with a `gpt-6-astra` thread and a
    `code-review high` pass, because a run that rises to high has a trigger in the diff,
    and no repeat of Step 3. Rerun after any change to the re-evaluation rule.
37. **Re-evaluation does not add a round review.** Setup: as item 36. Expected: Step 4
    has orchestrator findings only, and no Codex reviewer thread is recorded for Step 4;
    the only Codex calls in Step 4 are `codex-lite:implement` calls. Rerun after any
    change to Step 4 or the re-evaluation rule.
38. **Fable fallback.** Command: `/ccl:run #1 --effort max --no-codex`. Expected: the Step 3
    and Step 5 Codex slots are each a Fable subagent, or Opus with the Fable error
    recorded, and Step 5 still makes a `code-review xhigh` call in the tool trace. This
    confirms the Agent tool accepts the Fable model override in this session, that an
    error falls through to Opus, and that `--no-codex` does not remove the Claude pass.
    Rerun after any change to the fallback table or the session's model set.

## Environment checks

39. **Codex model ids.** Setup: a ChatGPT account with Codex, codex-lite 0.8.0 or later,
    and a scratch repository. Command: run a `/codex-lite:ask --model gpt-6.1-sol
    --timeout 60` call, then the same with `gpt-6-astra`, then a
    `codex-lite:implement` call whose request is `--model gpt-6-luna --timeout 60 --cwd
    <scratch repo>` on the first line and a one-line task to add a file on the next
    line, since the `--cwd` value is the rest of its line. Expected: all three return status `ok`, and the
    `implement` footer shows the new file. Rerun before each release and whenever a Codex
    call fails with a model error.
40. **Pre-approval of Codex calls.** Setup: default permission mode, and a command with
    `allowed-tools` limited to read-only `git` and `gh`. Command: `/ccl:run #1` at
    medium tier. Expected: record whether the codex-lite Bash call prompts, for `ask`,
    `review`, and `implement`. The default assumption is that it does. Rerun after any
    change to a command's `allowed-tools` or a Claude Code upgrade.

## M4: xhigh and max tier

Setup for items 41 to 54: a throwaway repo with a migration file, two areas of code that
share no file, and two open issues (#1, the common one-line bug, and #2, a second change
in the other area), and codex-lite 0.8.0 or later installed unless the item says
otherwise. An item that names an Opus-qualifying slice describes it in its setup, since
the common one-line bug never qualifies. Where an expectation names the model of an Agent
or Workflow call, or of a `codex-lite:implement` call, read it from the session's tool
trace, not from the plan or the report.

41. **Max tier reviews use `gpt-6-astra`.** Command: `/ccl:run #1 --effort max`.
    Expected: the report records a `gpt-6-astra` thread for Step 3 and another for Step
    5, a `code-review xhigh` pass per Step 5 round, a `codex-lite:implement` call at
    `gpt-6-astra` for the slice, and no Codex reviewer thread for Step 4. Rerun after any
    change to the roles table.
42. **A requested xhigh tier stands above the floor.** Command:
    `/ccl:run "add a column" --effort xhigh`. Expected: the run is xhigh, the report
    says the floor was applied and that the requested tier was above it, and both Step 3
    and Step 5 use `gpt-6-astra`, as xhigh always does, with a `code-review high` pass at
    Step 5. Rerun after any change to the risk floor or the trigger rule.
43. **Xhigh with a single-slice plan.** Command: `/ccl:run #1 --effort xhigh` for a
    one-line bug. Expected: the plan has one slice, Step 4 makes one
    `codex-lite:implement` call at `gpt-6-astra`, and the implementer is recorded as
    "codex", with the reason that no Opus criterion applies, and no Agent call for the
    slice. Rerun after any change to Step 4.2 or to the Implementer choice section of
    `tiers.md`.
44. **Re-evaluation does not raise an xhigh run.** Setup: a task sized xhigh whose
    implementation ends up removing an auth check. Command: `/ccl:run #1 #2`. Expected:
    the tier stays xhigh, the report says the floor applied at re-evaluation and the tier
    was unchanged, Step 3 and Step 5 used `gpt-6-astra` (xhigh always does, with or
    without a trigger), beside a `code-review high` pass. Rerun
    after any change to the re-evaluation rule or the trigger rule.
45. **Independent Codex slices at high tier run in series.** Setup: one cross-cutting
    change in one deliverable whose two parts touch disjoint files, each part about one
    subagent timeout of work, with no Opus criterion applying to either. Command:
    `/ccl:run #1 --effort high`. Expected: two `codex-lite:implement` calls at
    `gpt-6.1-sol`, the second not started before the first returns, no Workflow and no
    parallel calls, the serial choice logged in `run.md`, the tier still high, and a run
    log showing two round counters, one per slice. Rerun after any change to the Step 2
    slice rule, Step 4.2, or the Budgets section.
46. **Ordered slices at medium tier.** Setup: several files in one area, about two
    subagent timeouts of work in total, where the first slice creates a helper in a file
    it owns and the second slice, in files only it owns, calls that helper. Command:
    `/ccl:run #1 #2 --effort medium`. Expected: two `codex-lite:implement` calls at
    `gpt-6.1-sol` in order, the second not started before the first returns, the tier
    medium, and no file in both slices. Rerun after any change to the Step 2 slice rule or
    Step 4.2.
47. **Opus by the contract criterion at xhigh.** Setup: #1 asks for a new module in one
    area and #2 asks the other area to call it, so the plan has two slices and the first
    adds a module the second cites. Command: `/ccl:run #1 #2 --effort xhigh`. Expected:
    `opus` for the first slice in the plan and in the report, with the contract criterion
    named, and the Agent or Workflow call for that slice made with model `opus` in the
    tool trace, and the second slice a `codex-lite:implement` call at `gpt-6-astra`, run
    in series with it. Rerun after any change to the Implementer choice section of
    `tiers.md`.
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
50. **Codex chosen for a small max slice.** Setup: a max run whose second slice is one
    documentation file with no risk trigger and no new section that anything cites.
    Command: `/ccl:run #1 #2 --effort max`. Expected: a `codex-lite:implement` call at
    `gpt-6-astra` for that slice, recorded as "codex" in the plan and the report, and no
    Agent call for it at `sonnet` or `opus`. Rerun after any change to the Implementer
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
    `/ccl:run #1 #2 --effort xhigh`. Expected: the eight-file slice as a
    `codex-lite:implement` call at `gpt-6-astra` and the nine-file slice on `opus` with
    the file criterion named, in the plan, in the report, and in the tool trace. Rerun
    after any change to the Implementer choice section of `tiers.md`.
53. **A timeout is not a fallback.** Setup: the item 48 issue, so the slice qualifies for
    `opus`, and `.ccl.json` set to `{"timeouts": {"subagent": 1}}` so the call runs past
    its budget. Command: `/ccl:run #1 --effort xhigh`. Expected: the tool trace shows the
    implementer at `opus` and no later call at `sonnet`, and the run ends `blocked`
    naming the subagent budget. Rerun after any change to the Budgets section or the Opus
    fallback rule.
54. **Effective model after a Workflow.** Setup: #1 is the item 48 auth check in one
    area, and #2 is a change to a migration file in the other area that shares no file
    with it, so the plan has two independent slices and both qualify for `opus` by the
    risk criterion. The session makes an `opus` call error as in item 49, and the first
    slice's diff must draw a blocking finding so a Step 4.3 round needs a fresh agent.
    Command: `/ccl:run #1 #2 --effort xhigh`. Expected: with one Workflow, the tool trace
    shows one Workflow with both slices, its `opus` calls erroring, the reruns at
    `sonnet`, and the fresh agent at `sonnet`, and the report names the swaps. With
    parallel Agent calls in one message, the choice is logged in `run.md`, the `opus`
    calls error and are rerun at `sonnet`, and a continued agent, not a fresh one, is
    acceptable for the review round, still at `sonnet`, with the swaps in the report. No
    `codex-lite:implement` call appears, because neither slice is a Codex slice. Rerun
    after any change to Step 4.2 or the Opus fallback rule.

## M5: the second final reviewer

Setup for items 55 to 61: the M3 setup, with the built-in `code-review` skill listed in
the session unless the item says otherwise.

55. **High tier without a trigger uses `gpt-6.1-sol` and `code-review medium`.** Setup: a
    cross-cutting change inside one deliverable that touches no risk floor area. Command:
    `/ccl:run #1 --effort high`. Expected: `gpt-6.1-sol` threads for Step 3 and Step 5, a
    `codex-lite:implement` call at `gpt-6.1-sol`, one `code-review medium` call per Step 5
    round in the tool trace with the level passed explicitly and neither `--comment` nor
    `--fix`, the report's Claude review passes line filled per round, and the run log
    showing both passes returned before any Step 5 fix. Rerun after any change to Step 5,
    the Claude review contract, or the trigger rule.
56. **Floored high tier uses `gpt-6-astra` at both stages.** Command:
    `/ccl:run "add a column" --effort high`. Expected: a `gpt-6-astra` thread for Step 3,
    a `gpt-6-astra` thread for Step 5, a `code-review high` pass, and the implementer on
    `opus` by the risk criterion. Rerun after any change to the trigger rule.
57. **A missing `code-review` skill blocks only where it is needed.** Setup: a session in
    which the `code-review` skill is not listed. Each run gets its own branch name, since
    the first run's default branch would otherwise exist and block the last run at Step
    3.7.2 before it reaches Step 5. Command: `/ccl:run #1 --effort medium --branch t57-a`,
    then `/ccl:plan #1 --effort high`, then `/ccl:run #1 --effort low --branch t57-b`.
    Expected: the first ends `blocked` at the start of Step 5, after implementation,
    naming the missing skill; the second ends `plan-only` with no mention of the skill;
    and the third ends `blocked` at the start of Step 5, after implementation, naming the
    missing skill. Rerun after any change to the Claude review contract.
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
66. **Codex availability without the skill list.** Setup: codex-lite 0.8.0 or later
    installed and enabled, `codex` on PATH, and a session whose skill list omits
    `codex-lite:ask` and `codex-lite:implement`. Command: `/ccl:run #1`. Expected: Codex
    is treated as available, `run.md` records the check as `codex --version` and the
    plugin version, and the Step 3 call is attempted. If the Skill call errors because the
    skill is not listed, the call counts as `failed`: it is retried once with the same
    arguments, then the stage swaps to the Claude fallback, and the report names the swap
    with the reason "skill not listed in session", and Codex is recorded unavailable for
    the rest of the run, so the slice's implementer is `sonnet` and no
    `codex-lite:implement` call is attempted. Rerun after any change to Codex availability
    or the Reviewer contract.
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
    `/ccl:run #1 #2 --effort medium --no-codex`. Expected: both implementers, at `sonnet`
    by the `--no-codex` fallback, start in one message as parallel Agent calls, the choice
    and the reason are logged in `run.md`, and the review round goes to the same agent
    through SendMessage. Without `--no-codex` the same slices are Codex slices and run in
    series (item 45). Rerun after any change to Step 4.2 or 4.3.
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
    records it in `run.md`, works in it, runs Step 5 at low tier with an Opus subagent
    as the Claude slot and as the Codex slot, since `--no-codex` is set, and the
    report names the worktree path and `git worktree remove <path>`. The Step 0.1
    statement, printed before any worktree exists, already lists the `git worktree add`
    and `cd <checkout> && ...` prompts, and the report says the run was attended. Rerun
    after any change to Step 0.1, Step 0.3, or `worktree.md`.
73. **Skip-worktree files at medium tier.** Setup: as item 72, plus a `test` script that
    fails on the original tree's skip-worktree state and passes at the base commit, and
    codex-lite installed, and a lockfile and an install step the instruction files name
    (for example `npm ci`). Command: `/ccl:run #1 --effort medium`. Expected: the install
    step runs in the worktree before the baseline, the Step 5 diff review goes through
    `codex-lite:ask` with a patch file and no `codex-lite:review` call appears in the tool
    trace; the Claude slot is an Opus subagent given the worktree's diff and no
    `code-review` call appears; the slice is a `codex-lite:implement` call with `--cwd
    <worktree>` as its last option and a request that names files by absolute path or
    carries their content; the baseline and Step 6 checks run in the worktree, which the
    passing `test` shows; and the PR's head branch equals the branch the run created in
    the worktree. Rerun after any change to Step 0.3, Step 5.2, `worktree.md`, or the
    Reviewer contract.
74. **Skip-worktree files at high tier.** Setup: as item 72, and codex-lite installed.
    Command: `/ccl:run #1 --effort high`. Expected: the run is not `blocked` at Step 1.6;
    it works in the detached worktree, the Claude slot of Step 5 is an Opus subagent given
    the worktree's diff, recorded in `run.md` and named in the report as a substitute and
    not a swap, no `code-review` call appears in the tool trace, and the report names the
    worktree path. Rerun after any change to Step 0.3, Step 1.6, or `worktree.md`.
75. **Two-repo run at medium tier.** Setup: two throwaway GitHub repos on the same host,
    each with one open issue (#1 in each) that describes a one-line bug, the second
    checked out beside the first, and codex-lite installed. Command: run from the primary
    `/ccl:run #1 <URL of the second repo's issue> --repo <path of the second repo>`.
    Expected: one branch with the same name in each repo, one PR per repo with the sibling
    links filled by `gh pr edit`, `Closes` only from the issue's own repo and `Refs
    <owner>/<repo>#n` from the other, one comment per issue naming both PRs with the
    issue's own repo first, a Codex review of the primary by `codex-lite:review`, and of
    the second repo through `codex-lite:ask` with `diff-<slug>.patch`, the second repo's
    slice a `codex-lite:implement` call with `--cwd <second repo>` as its last option and
    a request that names files by absolute path, the second repo's PR opened against the
    second repo, its CI read from the second repo, and each PR body in
    `.ccl/<run-id>/pr-body-<slug>.md` in the primary, with every `--body-file` of a `gh`
    call for the second repo an absolute path. Rerun after any change to `multi-repo.md`
    or Step 7.
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
    an issue whose implementation removes an auth check, and codex-lite installed.
    Command: `/ccl:run #1 --effort medium`. Expected: Step 4.5 raises the tier to high and
    the run is not `blocked` there; Step 5 uses the trigger cell, a `gpt-6-astra` review
    through `codex-lite:ask` with a patch file, and an Opus subagent as the Claude slot,
    with no `code-review` call in the tool trace, and the report names the worktree path.
    Rerun after any change to Step 4.5, Step 0.3, or `worktree.md`.
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

Setup for items 83 to 127: the common setup, plus the setup each item names. Items 83 to
99 and 101 to 127 are hand runs against throwaway repos and cannot run inside a ccl run.
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
    switches to `t90`, and the second repo is `new`: it creates a branch from its default
    branch under the Step 3.7.2 naming rule, with the collision check, not named `t90`;
    the primary has no `gh pr create` and no body edit, and the second repo gets a new PR
    whose body says "pending" for its sibling link; the comment on the continued PR is
    posted only after the second repo's PR is open, with its "Related pull requests"
    section filled; `gh pr edit` covers only the second repo's PR; and each issue comment
    names both PRs. Rerun after any change to `multi-repo.md` or Step 7.2.
91. **An unnamed writable checkout leads to the Step 1.2 adoption question.** Setup: an
    issue whose text asks for a change in a second checkout on disk, for example "also
    update `/abs/path/other/README.md`", and a run with no `--repo`. Command: `/ccl:run
    #1 --no-codex`. Expected: Step 1.2 asks to adopt the toplevel of that checkout before
    Step 2, and a reply that is not `yes` or a path from the question ends `stopped`
    with the question and the rerun text in the report, nothing implemented and the
    checkout not adopted. The rerun text is the same inputs and flags plus `--repo
    <path>`, with the path from the issue, and, because neither `--branch` nor
    `--continue` was given, a block that names `--continue <branch>` and `@<branch>` and
    lists each checkout's `HEAD` as the run saw it, without choosing a branch. A reply
    of `yes` adopts it, as item 146. Then change the issue so the second checkout is only
    read: no question is asked and the run is not blocked there. Then ask for changes in
    two other checkouts, pass `--repo` for one of them, and reply with the path of the
    other only: the run asks about the unnamed one only, and a subset that leaves a
    flagged checkout missing ends `blocked` in Step 1.2 with the rerun command adding one
    `--repo` for the missing checkout only. Rerun after any change to Step 1.2.
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
    Codex, `code-review`, each check, each install step, each CI poll, each poll of a
    background check), right after each push returns, and at the terminal state; no time
    is logged for any other `git` call or for a substep; and each
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
104. **`--continue` with `HEAD` off the base commit asks to switch.** Setup:
     branch `t104` pushed, local `t104` absent or equal to the remote, and the session on
     the default branch with a clean tree. Command: `/ccl:run #1 --no-codex --continue
     t104`, then `/ccl:plan #1 --continue t104`. Expected: the Step 0.1 statement predicts
     the question, and in Step 0.2 each run asks "<path> is at <short sha> on <branch>.
     Switch to t104?" before anything is written. A non-answer ends `stopped` with nothing
     switched, written, or pushed. On `yes` the run switches with `git switch t104`, or
     `git switch -c t104 <remote>/t104` when local `t104` is absent, `run.md` records the
     previous `HEAD`, and the run continues. Then run `git switch --detach <remote>/t104`
     and rerun the first command: no question is asked, `run.md` records `HEAD` as the
     planning snapshot, and the plan review and the Step 1.3 reproduction read that code.
     Then add a `skip-worktree` edit so a worktree run is needed: with the session on
     `t104` the run ends `blocked` naming the session's checkout, and with the session
     detached at `<remote>/t104` it does not. Rerun after any change to Step 0.2 or
     `worktree.md`.
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
     open PR. Command: as item 75 with `--continue t108 --no-publish`. Expected:
     `prepared`; `.ccl/<run-id>/pr-body.md` and `.ccl/<run-id>/pr-body-<slug>.md` both
     exist and hold the continued-PR body of their repository; and the report gives one `gh pr
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
     edits a path in an unrelated git checkout that it names by absolute path: the Step
     1.2 adoption question, and a non-answer ends `stopped` with the rerun command in the
     report. Rerun after any change to Step 1.2.
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
115. **A repository that lacks the continued branch is `new`.** Setup: Multi-repo mode
     with a second checkout whose remote lacks branch `t115`, and the primary's remote
     has it. Command: `/ccl:run #1 --no-codex --continue t115 --repo <path>`. Expected:
     the primary continues `t115`, and the second repository is `new`: it gets the name
     the Step 3.7.2 naming rule produces, not `t115`, and the collision check runs for
     it. To continue a differently named branch there, pass `--repo <path>@<branch>`.
     Rerun after any change to Step 3.7.2 or `multi-repo.md`.
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
118. **The PR queries read open PRs first and cover more than 30 results.** Setup:
     branch `t118` with one open PR from this repository. Command: `/ccl:run #1
     --no-codex --continue t118`. Expected: the tool trace shows `gh pr list` with
     `--state open` and `--limit 100`, the run finds the open PR, and no `--state all`
     query runs. Then close that PR, and open and close 31 PRs from a fork's branch
     `t118`, each against its own base branch in this repository, so the 30 newest
     results are all from the fork. Expected: the `--state all` query runs with `--limit
     100`, finds this repository's closed PR, and Step 0.2 is a preflight failure naming
     it. Rerun after any change to Step 0.2.

119. **Each push is timed.** Command: `/ccl:run #1 --no-codex`. Expected: `run.md` holds
     a `date` time taken right after the push returned, and the CI watch does not judge
     CI until 2 minutes after that time. Rerun after any change to the Budgets section or
     `ci-watch.md`.
120. **A changed PR state before implementation or the first push ends `blocked`.**
     Setup: branch `t120` pushed with one open PR, and branch `t120-base` pushed from the
     default branch. Command: `/ccl:run #1 --no-codex --confirm-plan --continue t120`;
     when the plan is printed, close the PR, then reply "yes". Expected: `gh pr list`
     runs again after the reply, and the run ends `blocked` before Step 3.7.2 naming the
     closed PR, with nothing implemented and nothing pushed. Repeat with no PR at the
     start and a PR opened during the wait, again with a second PR opened, and again with
     the open PR retargeted to `t120-base` during the wait, and expect `blocked` each
     time. Then run without `--confirm-plan` and close the PR while Step 4 runs:
     `gh pr list` runs again before the push, and the run ends `blocked` with nothing
     pushed. Rerun after any change to Step 3.5 or Step 7.2.
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
     in the tool trace. Rerun after any change to Step 1.2 or Step 2.
124. **A `HEAD` that is off the branch is switched or fails with the command that works.**
     Setup: branch `t124` pushed, and a local `t124` one commit behind the remote.
     Command: `/ccl:plan #1 --continue t124` with the session on local `t124`. Expected:
     the switch question is asked, and on `yes` the run does not switch, because `HEAD` is
     already on it, and runs `git merge --ff-only <remote>/t124`. Then with the session on
     the default branch and local `t124` equal to the remote or absent: the question is
     asked, and on `yes` the run runs `git switch t124`, or `git switch -c t124
     <remote>/t124`. Then with local `t124` holding a commit that is not on the remote:
     a preflight failure whose message gives `git switch --detach <remote>/t124`, with no
     question. Rerun after any change to Step 0.2.
125. **A pull request list that may be incomplete is a preflight failure.** Setup: branch
     `t125` pushed with no PR from this repository, and 100 open PRs from a fork's branch
     `t125`, each against its own base branch in this repository (scripted with `git
     push` and `gh pr create --repo`). Command: `/ccl:run #1 --no-codex --continue
     t125`. Expected: a preflight failure in Step 0.2 saying the pull request list is
     incomplete, nothing written, and nothing pushed. Repeat with `--no-publish`: the run
     continues, `run.md` and the report record the incomplete list, and the report gives
     neither a `gh pr comment` nor a `gh pr create` command. Then add one open PR from
     this repository's `t125` and repeat with `--no-publish`: the same. Rerun after any
     change to Step 0.2 or Step 7.
126. **A file hidden during the `--confirm-plan` wait ends `blocked`.** Command:
     `/ccl:run #1 --no-codex --confirm-plan`; when the plan is printed, run `git
     update-index --skip-worktree <file>` on a tracked file, edit that file, then reply
     "yes". Expected: `blocked` before Step 3.7.2 naming the file, and nothing
     implemented. Rerun after any change to Step 3.5.
127. **A PR retargeted during the CI watch ends `blocked`.** Setup: a workflow that runs
     for at least 3 minutes on pull requests, and branch `t127-base` pushed from the
     default branch. Command: `/ccl:run #1 --no-codex`; after the PR opens, change its
     base branch to `t127-base`. Expected: the next poll ends
     `blocked` naming both branches, and the report does not say CI is green. Rerun after
     any change to `ci-watch.md`.

## M8: 0.7.0, 2026-09-30

Setup for items 128 to 141: the common setup, plus codex-lite 0.8.0 or later installed and
enabled, `codex` on PATH, and the built-in `code-review` skill listed, plus the setup each
item names. They are hand runs against throwaway repos. Read the model of a
`codex-lite:implement` call from its `--model` argument in the tool trace, not from the
plan or the report.

128. **Low tier implements with `gpt-6-luna` and reviews with both reviewers.** Command:
     `/ccl:run #1 --effort low`. Expected: `done`; the plan records the implementer as
     "codex"; the tool trace shows one `codex-lite:implement --model gpt-6-luna` call
     with `--timeout` in seconds and its request text on the line after the options, and
     no Agent call for the slice; Step 3 and Step 5 each use a `gpt-6.1-sol` thread;
     Step 5 makes one `code-review low` call per round, with the level and the range
     `<base-sha>...HEAD`; and the report's Step 5 reviewers line names both reviewers.
     Rerun after any change to the tier table, Step 4.2, or Step 5.
129. **Medium tier implements with `gpt-6.1-sol`.** Command: `/ccl:run #1 --effort
     medium`. Expected: one `codex-lite:implement --model gpt-6.1-sol` call for the slice,
     recorded as "codex" in the plan, a `gpt-6.1-sol` thread for Step 5, and a
     `code-review medium` pass per round. Rerun after any change to the tier table or the
     Implementer choice section of `tiers.md`.
130. **High tier implements with `gpt-6.1-sol`, and Opus only by the criteria.** Setup: a
     task that meets no Opus criterion, and then the item 48 auth check. Command:
     `/ccl:run #1 --effort high` for each. Expected: the first is one
     `codex-lite:implement --model gpt-6.1-sol` call and no Agent call for the slice; the
     second has the slice on `opus` by the risk criterion, an Agent call at model `opus`
     and no `codex-lite:implement` call for it, and its final review takes the trigger
     cell, `gpt-6-astra` with `code-review high`. Rerun after any change to the
     Implementer choice section of `tiers.md` or the trigger rule.
131. **Xhigh and max implement with `gpt-6-astra`, and their final review does not depend
     on a trigger.** Setup: a task with no risk floor trigger. Command:
     `/ccl:run #1 --effort xhigh`, then `/ccl:run #1 --effort max --branch t131-b`.
     Expected: each has one `codex-lite:implement --model gpt-6-astra` call, recorded as
     "codex" in the plan; a `gpt-6-astra` thread for Step 3 and another for Step 5; and
     `code-review high` at xhigh and `code-review xhigh` at max, although no trigger was
     present at the estimate or in the diff. Rerun after any change to the tier table or
     the trigger rule.
132. **A Step 6 fix and a CI repair at low tier go through a Step 5 round.** Setup: a
     low-tier task whose fix makes a check fail that the baseline passed, so a Step 6 fix
     is needed, and then a change that passes locally and fails one CI job the loop can
     fix in one edit. Command: `/ccl:run #1 --effort low` for each. Expected: the Step 6
     fix is followed by a Step 5 round with a Codex pass and a `code-review low` pass over
     a diff that includes it, inside Step 5's cap of 3, and the CI repair cycle uses the
     Step 5 Codex thread, or `codex-lite:review --base <base-commit>` when none exists,
     beside the Claude pass, with one extra review round per cycle and not the
     orchestrator's own review. Rerun after any change to Step 6.5 or Step 7.3.5.
133. **A fix round for a Codex slice is a fresh `implement` call.** Setup: a slice whose
     diff must draw a blocking finding in Step 4, at medium tier, and a second run where
     the finding comes from Step 5. Command: `/ccl:run #1 --effort medium` for each.
     Expected: each fix is a new `codex-lite:implement` call with the findings and the
     slice's current diff in its request, no `--resume` in any `implement` call, no
     SendMessage to a Codex implementer, and a new thread id in `run.md` for each call.
     Rerun after any change to Step 4.3 or Step 5.3.
134. **Two failed Codex implementer calls fall back to `sonnet`.** Setup: a session in
     which `codex-lite:implement` returns `status: failed` twice in a row on a POSIX
     system, for example a Codex login that fails on the call, having left part of the
     slice in the tree. Command: `/ccl:run #1 --effort medium`. Expected: between the
     calls, the output of the failed call has returned and `run.md` records the tree
     state read from its footer or `git status`; the second `implement` call carries the
     current diff with the same slice prompt; after the second failure the same prompt
     goes to an Agent call at `sonnet`, which also has the current diff; the slice's
     effective model is `sonnet`; and the report names the implementer swap. Rerun after
     any change to Approval scope item 6 or the implementer fallback rule.
135. **A call that may still be running ends the run `blocked`.** Setup: a session in
     which a `codex-lite:implement` call prints "codex may still be running as pid", and
     a second in which the call's output never arrives before its budget. Command:
     `/ccl:run #1 --effort medium` for each. Expected: the run ends `blocked`, naming the
     running process in the first and the budget expiry in the second, and no retry, no
     Sonnet call, and no other write to that checkout follows in the tool trace. Rerun
     after any change to Approval scope item 6.
136. **A `failed` or cut-off implementer call on Windows is not retried.** Setup:
     Windows, with a `codex-lite:implement` call that returns `status: failed` and prints
     no warning about running processes; then, in a second run, one whose output is cut
     off with no `status:` line at all (a Bash timeout with background tasks disabled, as
     codex-lite's acceptance item 14 sets up). Command: `/ccl:run #1 --effort medium`,
     each time. Expected: both runs end `blocked` naming the possible surviving process;
     there is no second `implement` call and no Agent call at `sonnet` for the slice. A
     Skill call that errors because `codex-lite` is not listed is the one exception and
     follows item 7 of Codex availability. Rerun after any change to Approval scope item 6.
137. **`--no-codex` and a missing Codex give `sonnet` implementers.** Command:
     `/ccl:run #1 --effort xhigh --no-codex`, then, with `codex` removed from PATH,
     `/ccl:run #1 --effort low --branch t137-b`. Expected: in both runs the slice goes
     to an Agent call at `sonnet`, the plan recorded "codex" and the report names the
     implementer swap with its reason, and no `codex-lite:implement` call appears. Rerun
     after any change to the fallback rules or Step 0.6.
138. **A `refused` implementer is `blocked` and a `timeout` is a budget expiry.** Setup: a
     session in which `codex-lite:implement` returns `status: refused`, and then a run
     with `.ccl.json` `{"timeouts": {"subagent": 1}}` so the call returns `status:
     timeout`. Command: `/ccl:run #1 --effort medium` for each. Expected: the first ends
     `blocked` with the message and no retry and no swap; the second ends `blocked`
     naming the implementer `--timeout`, with no retry and no swap. Rerun after any change
     to the Budgets section or the implementer fallback rule.
139. **The implementer budget is capped at 3600 seconds and logged.** Setup: `.ccl.json`
     with `{"timeouts": {"subagent": 90, "codex": 90, "run": 300}}`. Command:
     `/ccl:run #1 --effort medium`. Expected: the `codex-lite:implement` call passes
     `--timeout 3600`, `run.md` records the cap and the value passed, every reviewer
     Codex call passes `--timeout 3600` from the Codex budget capped at 60 minutes, and
     an Opus or Sonnet Agent implementer, in a run that has one, keeps the full 90
     minutes. Rerun after any change to the Budgets section.
140. **The orchestrator installs a dependency the plan adds.** Setup: an issue that needs
     a new package, a lockfile, and a slice whose check needs the network. Command:
     `/ccl:run #1 --effort medium`. Expected: after the Step 3.7.3 baseline ran on the
     unchanged base and after any ask-first approval, the orchestrator installs the
     package before the first implementer starts, the manifest and lockfile edits are in
     the Step 5 diff, the implementer makes no install attempt, and the orchestrator runs
     the network check after the implementer returns, sending a failure back as a
     finding. Rerun after any change to Step 3.7 or the Implementer prompt.
141. **A worktree run uses the Opus substitute at every tier.** Setup: as item 72.
     Command: `/ccl:run #1 --effort low`, then `/ccl:run #1 --effort max --branch t141-b`.
     Expected: neither run is `blocked` for its tier; each Step 5 has an Opus subagent as
     the Claude slot, recorded in `run.md` and named in the report as a substitute, with
     no `code-review` call in the tool trace; the Codex slice is a `codex-lite:implement`
     call whose last option is `--cwd <worktree path>` and whose request names files by
     absolute path or carries their content. Rerun after any change to `worktree.md` or
     the Claude review contract.

## M9: 0.8.0, 2026-09-30

Setup for items 142 to 167: the common setup, plus codex-lite 0.8.0 or later installed and
enabled, plus the setup each item names. Items that name codex-lite 0.9.0 need it. They
are hand runs against throwaway repos. Items 146 to 157 and 161 use the setup of item 75
(a primary and a second checkout `other`, each with a bare remote) unless they say
otherwise, and a prose input names `other` by absolute path and asks for a change in it,
with no `--repo` unless the item passes one. "The question" is the in-session question the item names, and a reply
is typed in the session. A non-answer is any reply that is not one of the question's
clear answers, for example "what does that mean?".

142. **Credentials: `keep` writes and forwards them.** Setup: a handoff file with a line
     `api_key: sk-test-0001` and a line `password=hunter2`. Command: `/ccl:run
     handoff.md --no-codex`. Expected: the Step 0.1 prediction lists the credentials
     question first, before the prompt; the question lists `api_key` and `password` by key
     name and line number and shows neither value; after `keep`, `inputs.md` holds both
     values, `run.md` records the decision and the two `date` times taken before and
     after the question, and no file was written before Step 0.5. Rerun after any change
     to Step 0.1 or Step 0.1a.
143. **Credentials: `drop` redacts every artifact and request.** Setup: as item 142, at
     medium tier with Codex. Command: `/ccl:run handoff.md --effort medium`, reply `drop`.
     Expected: `inputs.md` holds `<redacted: api_key>` and `<redacted: password>` in place
     of the values; a search of `.ccl/<run-id>/`, every Codex request file, every
     implementer prompt in the tool trace, and every patch finds neither `sk-test-0001`
     nor `hunter2`; and `run.md` records `drop`. Rerun after any change to Step 0.5 or
     the reviewer and implementer prompts.
144. **Credentials: no match asks nothing.** Setup: a handoff file with the words
     "token" and "password" in prose only, with no `key: value`, `key=value`, or
     `"key": "value"` form with a value, and no AWS key id or PEM header. Command:
     `/ccl:run handoff.md --no-codex`. Expected: no credentials question, none predicted
     in the Step 0.1 statement, and `run.md` records no credentials decision. Rerun after
     any change to the scan shapes.
145. **Credentials: a non-answer ends `stopped`.** Setup: as item 142. Command: `/ccl:run
     handoff.md --no-codex`, reply "continue". Expected: `stopped`, the report holds the
     question, with key names and line numbers and no value, no `.ccl/` directory holds a
     credential value, and nothing else ran. Rerun after any change to Step 0.1a.
146. **Repositories: `yes` adopts the checkout.** Setup: a prose handoff file naming
     `other` by absolute path and asking for a change in it. Command: `/ccl:run
     handoff.md --no-codex`, reply `yes` to the repository question and, to the branch
     question, `<primary path>@new` and `<other path>@new`, one per line. Expected: the
     question lists the toplevel of `other`, and the branch question lists "no
     suggestion" for both repositories, because both checkouts are on default branches,
     so a bare `yes` is not a valid reply; both end in state `new`; the run enters
     Multi-repo mode with `other` as an additional repository,
     `run.md` records "Multi-repo mode adopted at Step 1.2 by reply", Host detection,
     instruction files, default branch, and clean tree ran for `other` and the permission
     statement was printed again with `other`'s push and PR, and the run is not `blocked`.
     Rerun after any change to Step 1.2 or the late-entry order in `multi-repo.md`.
147. **Repositories: a subset that leaves a repository missing ends `blocked`.** Setup:
     as item 146, with a third checkout `third` that the handoff also asks to change.
     Command: `/ccl:run handoff.md --no-codex`, reply with the path of `other` only.
     Expected: `blocked` in Step 1.2 before Step 2, nothing implemented, and the report
     gives the rerun command with `--repo <path>` for `third`, and the branch block of
     item 91, because no `--branch` or `--continue` was given. Rerun after any change to
     Step 1.2.
148. **Repositories: a non-answer ends `stopped`.** Setup: as item 146. Command: as item
     146, reply "which one is that?". Expected: `stopped`, the report holds the question
     and the rerun text, `other` is not adopted, and nothing in `other` was touched.
     Rerun after any change to Step 1.2.
149. **Adoption is `blocked` when the primary is a worktree run.** Setup: as item 146,
     with the primary clean in status and index and a skip-worktree file differing from
     `HEAD`, so Step 0.3 creates a worktree. Command: `/ccl:run handoff.md --no-codex`.
     Expected: `blocked` before the repository question is asked, nothing in `other`
     touched, and the report says the rerun needs the skip-worktree edits cleared and the
     `--repo` flags. Rerun after any change to Step 1.2 or Step 0.3.
150. **Adoption is `blocked` when an adopted repository is dirty.** Setup: as item 146,
     with an uncommitted edit in `other`. Command: `/ccl:run handoff.md --no-codex`, reply
     `yes`. Expected: `blocked` naming `other` as dirty, no stash or checkout in the tool
     trace, the edit in `other` intact, and no branch question asked. Rerun after any
     change to the late-entry order in `multi-repo.md`.
151. **Branches: `yes` takes one suggestion per repository.** Setup: as item 146, with the
     primary on branch `t151` at its remote tip and `other` on branch `t151-etl` at its
     remote tip, both pushed. Command: `/ccl:run handoff.md --no-codex`, reply `yes` to
     the repository question and `yes` to the branch question. Expected: the question
     prints "<path>: suggested t151" for the primary and "<path>: suggested t151-etl" for
     `other`; each repository's `run.md` state is `continue` with that branch; no switch
     happens because each is at its tip; and the base commit of each is its remote branch
     head. Rerun after any change to the branch question in Step 1.2.
152. **Branches: a repository with two suggestions needs a line.** Setup: as item 151,
     plus a third checkout `third` on an unrelated branch at its remote tip and a remote
     branch `t151-heart` on `third`, so it has two suggestions. Command: as item 151 with
     `third` in the handoff, reply `yes` alone. Expected: the reply is not clear, because
     it does not cover `third`, and the run ends `stopped` with the question in the
     report. Then rerun and reply `yes` and `<third path>@t151-heart` on the next line:
     all three repositories continue the stated branches, and `third` is switched
     (item 155). Rerun after any change to the branch question.
153. **Branches: `@new` creates a branch in one repository.** Setup: as item 151. Command:
     as item 151, reply `yes` and `<other path>@new`. Expected: the primary continues
     `t151`, `other` is in state `new`, and a new branch is created in `other` under
     Step 3.7.2's naming rule, not the primary's branch name. Rerun after any change to
     the branch question or Step 3.7.2.
154. **Branches: a non-answer ends `stopped`.** Setup: as item 151. Command: as item 151,
     reply `yes` to the repository question and "whatever you think" to the branch
     question. Expected: `stopped` with the question in the report, no switch, no branch
     created, and no file edited. Rerun after any change to the branch question.
155. **A clean checkout behind its branch is switched with consent.** Setup: as item 146,
     `other` clean on an unrelated branch, with branch `t155` pushed to its remote and a
     local `t155` one commit behind it. Command: `/ccl:run handoff.md --no-codex
     --repo <other path>@t155`, reply `yes` to the switch question. Expected: the Step 0.1
     statement predicts the switch question; the question reads "<path> is at <short sha>
     on <branch>. Switch to t155?"; after `yes` the tool trace has `git -C <other path>
     switch t155` followed by `git -C <other path> merge --ff-only <remote>/t155`;
     `run.md` records the previous `HEAD`; the clean-tree check reruns; the report lists
     the switch under "Where the work is"; and the run ends without switching back. A
     second run with the local `t155` absent uses `git switch -c t155 <remote>/t155`. A
     third run with the session detached at `<remote>/t155` and the local `t155` one
     commit behind still asks, and after `yes` fast-forwards `t155`, so Step 3.7.2 does
     not switch to a stale branch. Rerun after any change to Step 0.2 or Step 0.1 item 3.
156. **A divergent local branch still fails preflight.** Setup: as item 155, with local
     `t155` holding a commit the remote lacks as well as lacking one the remote has.
     Command: as item 155. Expected: preflight fails with the 0.7.0 message for a
     divergent local branch, no switch question is asked, and no `git switch` runs. Rerun
     after any change to Step 0.2.
157. **A dirty checkout still fails preflight.** Setup: as item 155, with an uncommitted
     edit in `other`. Command: as item 155. Expected: `blocked` or preflight failure
     naming `other` as dirty, no switch question, no `git switch`, no stash, and the edit
     intact. Rerun after any change to Step 0.2 or Step 0.3.
158. **`--repo <path>@<branch>` is accepted without `--continue`.** Setup: as item 75,
     with branch `t158` pushed to `other`'s remote. Command: `/ccl:run #1 --no-codex
     --repo <other path>@t158`. Expected: the command does not reject the value; the
     invocation block has `repos: <other path>@t158`; the primary's state is `new` with a
     new branch and `other`'s is `continue t158`; and no `--continue` was needed. Rerun
     after any change to `commands/run.md` or `commands/plan.md` step 2.
159. **A missing explicit branch fails preflight.** Setup: as item 158 with no `t159` on
     `other`'s remote. Command: `/ccl:run #1 --no-codex --repo <other path>@t159`.
     Expected: the command accepts the value and loads the skill, and Step 0.2 fails
     preflight naming the missing branch and that repository's selected remote, with
     nothing written and the branch not created. A right part that passes the charset
     rule but fails `git check-ref-format --branch`, such as `@a..b`, splits and is
     rejected by the command before Step 0 as an invalid branch; a right part that fails
     the charset rule, such as `@a~b` or `@a b`, means no split, and the value is
     rejected before Step 0 as a missing directory. Rerun after any change to the
     commands' `--repo` rules or Step 0.2.
160. **A path containing `@` with no branch is accepted.** Setup: a checkout at a
     directory whose name contains `@`, for example `other@v2`. Command: `/ccl:run #1
     --no-codex --repo <path to other@v2>`. Expected: the whole value is taken as a path,
     no split happens, the run proceeds with that repository in state `new`, and the
     invocation block holds the full path. Rerun after any change to the `--repo` split
     rule.
161. **`prepared` gives each repository its own branch.** Setup: three repositories on a
     non-GitHub host (local bare remotes, as item 62), the primary and two additional,
     each with a diff. Branch `t161-a` is on the primary's remote and `t161-b` on the
     second's, and the third's remote lacks `t161-a`, so its state is `new`. Command:
     `/ccl:run "rename the README heading" --no-codex --continue t161-a --repo
     <b path>@t161-b --repo <c path>`. Expected: Step 7 never runs, the run ends
     `prepared`, and the report gives `git push <remote> t161-a`, `git push <remote>
     t161-b`, and `git push <remote> <the new branch>`, one per repository with a diff,
     each from that repository, and no push ran. Rerun after any change to Terminal
     states or Step 3.7.2.
162. **`review --cwd` reviews each additional repository at codex-lite 0.9.0, and the
     patch rule applies below it.** Setup: item 75 at medium tier with codex-lite 0.9.0,
     and a second session with codex-lite 0.8.0. Command: `/ccl:run #1 <URL of the second
     repo's issue> --effort medium --repo <other path>`, in each. Expected: at 0.9.0 the
     Step 5 trace has a `codex-lite:review --base <other's base> --model <id> --timeout
     <s>` call with `--cwd <other path>` on its second line for `other`, a separate
     thread from the primary's, `run.md` records a thread id per repository, and
     `diff-<slug>.patch` is still written; at 0.8.0 `other` is reviewed through `ask`
     with the patch and no `review --cwd` call appears. Rerun after any change to
     `multi-repo.md` Step 5.2 or
     the Reviewer contract.
163. **A follow-up resumes the right repository's thread.** Setup: as item 162 at 0.9.0,
     with a defect planted in `other` that its reviewer finds and one in the primary.
     Command: `/ccl:run #1 <URL of the second repo's issue> --effort medium --repo
     <other path>`. Expected: the Step 5.4 follow-up for `other` is an `ask --resume` on
     `other`'s thread id with `diff-<slug>.patch`, the primary's follow-up uses the
     primary's thread, and no call resumes another repository's thread; a CI repair in
     `other` does the same. Rerun after any change to `multi-repo.md` Step 5.4 or
     Step 7.3.5.
164. **Credentials: `drop` with a credential in diff context redacts the review.** Setup:
     as item 143 (medium tier with Codex), plus a tracked file in the primary, for
     example `config.yml`, holding `token: tk-test-0002` on the line next to one the task
     edits. Command: `/ccl:run handoff.md --effort medium`, reply `drop`. Expected: the
     Step 5 trace has no native `codex-lite:review` call for the primary; it has a fresh
     `codex-lite:ask` thread naming `.ccl/<run-id>/diff.patch`, which holds `<redacted:
     token>` in place of the value; `run.md` records the substitution and that thread as
     the stage's thread; follow-ups resume it; and no Codex request file or patch holds
     `tk-test-0002`. A second run with no credential in the diff makes the native review
     as usual. Rerun after any change to Reviewer contract item 5 or `multi-repo.md`
     Step 5.2.
165. **A change during the switch question's wait ends `blocked`.** Setup: as item 155.
     Command: as item 155; before replying `yes`, add an untracked file in `other`.
     Expected: `blocked` naming the untracked file in `other`, and no `git switch` or
     `merge` in the tool trace. A second run that instead commits on local `t155` during
     the wait, so it is ahead or divergent, ends `blocked` naming the local branch. Rerun
     after any change to Step 0.2 or the Questions mechanic.
166. **A predicted worktree run leaves the session's files alone.** Setup: a single
     repository with branch `t166` pushed and a local `t166` one commit behind it, the
     session on the default branch with a clean status and index, and a skip-worktree
     file with a local edit whose committed content differs between `t166` and the
     default branch. Command: `/ccl:run #1 --no-codex --continue t166`, reply `yes`.
     Expected: the question asks only to fast-forward local `t166`; the trace has
     `git fetch . refs/remotes/<remote>/t166:refs/heads/t166` and no `git switch` of any
     form in the session's checkout; the skip-worktree file keeps its local edit; Step
     0.3 creates the worktree at `<remote>/t166`; the `worktree.md` branch check passes;
     and Step 3.7.2 switches to `t166` inside the worktree. A second run with the session
     on the behind local `t166` asks to detach and fast-forward, and after `yes` the
     trace has `git switch --detach` with no commit argument, then the fetch, and the
     session is left detached at its previous commit. A third run with the session on
     `t166` at its remote tip asks only to detach and continues the same way. Rerun
     after any change to Step 0.2, Step 0.3, or `worktree.md`.
167. **Branches: a sibling name with a shared stem is suggested.** Setup: as item 146,
     with a third checkout `third`, with its own bare remote, that the handoff also asks
     to change. Push `feature/u/2026.09-mig-fixes` in the primary,
     `feature/u/2026.09-mig-heart`, `feature/u/9999-unrelated`, `feature/u/0001-noise`,
     and `feature/u/2026.10-later` in `other`, and `feature/u/2026.09-mig-etl` in
     `third`. The primary is on `feature/u/2026.09-mig-fixes`, `third` on
     `feature/u/2026.09-mig-etl`, and `other` on `feature/u/9999-unrelated`, each at its
     remote tip. Command: `/ccl:run handoff.md --no-codex`, reply `yes` to the
     repository question. Expected: the branch question prints "<path>: suggested
     feature/u/2026.09-mig-fixes" for the primary, "<path>: suggested
     feature/u/2026.09-mig-etl" for `third`, and for `other` one line naming
     `feature/u/9999-unrelated`, `feature/u/2026.09-mig-heart`, and
     `feature/u/2026.10-later`, so it has three suggestions, and not
     `feature/u/0001-noise`, which shares only the directory. `feature/u/2026.10-later`
     shares the stem `feature/u/2026` and is the intended over-suggestion. A
     reply of `yes` alone is not clear, because it does not cover `other`, and the run
     ends `stopped` with the question in the report. Then rerun and reply `yes` and
     `<other path>@feature/u/2026.09-mig-heart` on the next line: all three repositories
     continue the stated branches, and `other` is switched with the reply as consent
     (item 155). Rerun after any change to the branch question.

## M10: 0.9.0, 2026-10-01

Setup for items 168 to 176: the common setup, plus the cca plugin, 0.2.0 or later,
installed for the items that name it. Items 168 to 176 are not yet run and have no entry
in the record of runs. Item 168 is the check that fails if the handoff mapping breaks: it
is run with cca's `handoff.sh check`. Items 173 and 175 need a second input or a second
issue, as they say. Item 176 needs item 174's setup.
"The handoff" is `.ccl/<run-id>/handoff.md`, and "the manifest" is
`.ccl/<run-id>/cca-manifest.json`. "The audit line" is the report's `Audit:` header line.

168. **A `done` one-input, one-bundle run under 500 changed lines writes a handoff cca
     accepts and no audit line.** Not yet run. Setup: the common setup, with a fix that
     changes fewer than 500 lines. Command: `/ccl:run #1 --no-codex`, then run cca's
     `handoff.sh check` on the handoff. Expected: the run ends `done`; the handoff and the
     manifest exist; the check accepts the handoff with no error; the manifest names one
     bundle with the PR as `github:<owner>/<repo>#<n>` and `claims` the handoff's absolute
     path; `Handoff:` gives the path; `Audit:` reads "not suggested". Rerun after any
     change to `handoff.md`, `report.md`, or Final report handling.
169. **The same run at 500 or more changed lines writes the audit line.** Not yet run.
     Setup: as item 168, with a change of 500 or more lines added plus deleted over the
     three-dot diff from the base. Command: `/ccl:run #1 --no-codex`. Expected: as item
     168, and `Audit:` reads `/cca:audit "<absolute path of cca-manifest.json>"` and says
     it needs the cca plugin and that the bounds are a suggestion. Rerun after any change
     to the audit bounds in `report.md`.
170. **A `prepared` run whose push was declined after Step 7.1 committed writes both
     files.** Not yet run. Setup: the common setup, default permission mode. Command:
     `/ccl:run #1 --no-codex`, and decline the push when it is asked. Expected: the run
     ends `prepared` with the Step 7.1 commit present; the handoff and the manifest exist;
     the manifest has the bundle's `branch` and `base` and no `pr`; the handoff's
     `pr` is `none`. Rerun after any change to Step 7.1 or the `handoff.md` timing rule.
171. **A `done` two-input run writes the audit line, and cca's stage 1 passes.** Not yet
     run. Setup: the common setup with a second open issue (#2) that describes another
     small bug. Command: `/ccl:run #1 #2 --no-codex`, then `/cca:audit "<absolute path of
     cca-manifest.json>"`. Expected: the handoff has two tickets; `Audit:` is filled
     because there is more than one ticket; the cca audit reads the manifest and passes its
     stage 1. Rerun after any change to the manifest fields.
172. **A `--no-publish` run writes neither file and says why.** Not yet run. Setup: the
     common setup, with uncommitted changes left by the run. Command: `/ccl:run #1
     --no-codex --no-publish`. Expected: no `handoff.md` and no `cca-manifest.json` under
     `.ccl/<run-id>/`; `Handoff:` reads "not written" with "no commit from this run"; the
     report says `/cca:handoff` in this session can write one after you commit; `Audit:`
     reads "not suggested"; no commit was made to make a handoff possible. Rerun after any
     change to the `handoff.md` timing rule.
173. **The same issue given as `#n` and as its URL gives one ticket.** Not yet run.
     Setup: the common setup. Command: `/ccl:run #1 <URL of issue 1> --no-codex`.
     Expected: the handoff has one ticket, `github:<owner>/<repo>#1`, and the manifest
     lists that id once under `tickets`; the ticket's `iteration` and `owner` match the
     issue's milestone title and first assignee, or `none`. Rerun after any change to
     input parsing or the ticket rules.
174. **A Multi-repo run with different branches per repository gives one bundle each.**
     Not yet run. Setup: item 75's two throwaway GitHub repos on the same host, each with
     a commit-worthy change. The branch `t174-b` is created in the second repo and pushed
     to its GitHub remote beforehand, and both checkouts are clean at their remote tips
     before the run. `other` continues `t174-b` through `--repo`. Command: `/ccl:run #1
     <URL of the second repo's issue> --no-codex --repo <other path>@t174-b`. Expected:
     the handoff has two bundles, the primary first, each with its own `repo`, `pr`,
     `branch`, and `base`; the manifest has the same two bundles; each `base` is
     `<remote>/<base branch>` for that repository's remote; names follow the naming rule,
     with `-2` on a clash. Rerun after any change to the bundle rules.
175. **An input with no commit lists the first bundle and `commits: none`.** Not yet run.
     Setup: the common setup, with two inputs where one, a quoted description, asks for
     nothing the diff changes, so no commit message names it. Command: `/ccl:run #1
     "note that the README is current" --no-codex`. Expected: the ticket
     `<run-id>/input-2` lists the first bundle and has `commits: none`; the other ticket
     lists its commits; `cca`'s `handoff.sh check` accepts the handoff. Rerun after any
     change to the ticket mapping.
176. **A Multi-repo run does not claim a check that failed in the ticket's first bundle's
     repository.** Not yet run. Setup: as item 174, with one check command (for example a
     `test` script) present in both repos, passing in the primary and failing at baseline
     in the second repo, where the failure is allowed. Command: as item 174. Expected: the
     ticket for #1 lists that command in `verified` as passed, naming the primary's bundle
     and its absolute directory; the ticket for the second repo's issue, whose first
     bundle is the second repo's, does not list it as passed, and its `verified` is `none`
     or lists only checks that passed in the second repo; cca's `handoff.sh check`
     accepts the handoff. Rerun after any change to the `verified` rule.

## M11: 0.10.0, 2026-10-02

Setup for items 177 to 181: the common setup, and cca installed at the version each item
names, or not installed when it says so. Items 177 and 180 also need a GitHub issue that
has a parent issue. Items 180 and 181 use the fixtures they name. Items 177 to 181 are
not yet run and have no entry in the record of runs. Every item but 179 needs cca 0.3.0,
which is not yet released.

177. **A run on an issue with a parent and a closing PR writes both keys with cca 0.3.0
     and neither with cca 0.2.0.** Not yet run. Setup: the common setup, with an issue
     that has a parent and a change whose run opens a PR that closes it. Install cca
     0.3.0 for the first run and cca 0.2.0 for the second. Command:
     `/ccl:run #<n> --no-codex`, run once with cca 0.3.0 and once with cca 0.2.0, then run
     cca's `handoff.sh check` on each handoff. Expected: with cca 0.3.0 the run's ticket
     has `parent: github:<owner>/<repo>#<parent>` and `links:` with a `closed by:` entry
     for the run's own PR, both right after `owner`, and cca's `handoff.sh check` accepts
     the handoff, and the tool trace shows the parent query with `--hostname github.com`;
     with cca 0.2.0 the ticket has neither key, the check accepts it, the tool trace shows
     neither read, and the `Handoff:` line says the gate left them out with the installed
     version. Rerun after any change to the version gate, the reads, or the ticket mapping
     in `handoff.md`.
178. **A successful empty read writes `links: none` and no `parent`.** Not yet run.
     Setup: the common setup with cca 0.3.0; issue #1 has no parent, and no PR closes it.
     Command: `/ccl:run #1 --no-codex`, and decline the push when it is asked, so the run
     ends `prepared` with its Step 7.1 commit and opens no PR. Expected: `run.md` records,
     for #1, no parent and no closing PRs, with no failed read; the ticket has
     `links: none` right after `owner` and no `parent` key; cca's `handoff.sh check`
     accepts the handoff; the `Handoff:` line names no failed read. Rerun after any change
     to the reads or the ticket mapping in `handoff.md`.
179. **With no cca installed, no read is made and the report says why.** Not yet run.
     Setup: the common setup with cca uninstalled, so `claude plugin list --json` has no
     `cca@` entry. Command: `/ccl:run #1 --no-codex`. Expected: the run ends `done`; the
     tool trace shows neither the `closedByPullRequestsReferences` read nor the
     `gh api graphql` parent query; `run.md` records once that the gate skipped the reads
     because no `cca@` entry was found; the ticket has neither key; the `Handoff:` line
     says the gate left them out because no `cca@` entry was found, and that cca's check
     was not run because the plugin is absent. Rerun after any change to the version gate
     in `handoff.md`.
180. **Each read can fail on its own, and a failed read writes no key.** Not yet run.
     Setup: the common setup with cca 0.3.0 and an issue #<n> that has a parent. Fixture:
     a script named `gh` in a directory placed first on `PATH` before Claude Code starts.
     It exits 1 with a one-line error for one read and passes every other call, unchanged,
     to the real `gh`. In the first run it fails a call whose arguments include
     `closedByPullRequestsReferences`. In the second it fails a `gh api graphql` call whose
     query contains `parent{`. Command: `/ccl:run #<n> --no-codex`, once per fixture
     setting. Expected, first run: the ticket has `parent` and no `links` key; `run.md`
     records the closing-PR read as failed with the error line; the `Handoff:` line names
     the ticket and that `links` was left out. Second run: the ticket has `links` with a
     `closed by:` entry for the run's own PR and no `parent` key; `run.md` records the
     parent read as failed with the error line; the `Handoff:` line names the ticket and
     that `parent` was left out. Neither run writes `links: none`, and cca's
     `handoff.sh check` accepts both handoffs. Rerun after any change to the reads, the
     failed-read rule, or the `Handoff:` rule.
181. **The gate compares versions as numbers and fails closed on an unparsable one.** Not
     yet run. Setup: the common setup. Fixture: a local copy of cca 0.3.0, installed from
     a local marketplace in place of cca, with the `version` in its
     `.claude-plugin/plugin.json` set to `0.10.0` for the first run and to a commit hash,
     such as `0123456789ab`, for the second. Command: `/ccl:run #1 --no-codex`, once per
     version. Expected, `0.10.0`: the gate passes; the tool trace shows both reads; the
     ticket has `links` with a `closed by:` entry for the run's own PR and no `parent`,
     since #1 has none; cca's `handoff.sh check` accepts the handoff. Commit hash: the gate
     fails; the tool trace shows neither read; `run.md` records once that the gate skipped
     the reads, with the installed version; the ticket has neither key; the `Handoff:` line
     says the gate left them out, with the installed version. Rerun after any change to the
     version gate in `handoff.md`.

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
