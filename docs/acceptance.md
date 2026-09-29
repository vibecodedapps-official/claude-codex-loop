# Acceptance checks

The plugin is prompt-only and has no automated test surface in 0.1.0, so these checks
are run by hand against a throwaway repo. Each item gives the setup, the command, the
expected result, and when to rerun it. The record of runs is at the end.

Common setup for items 4 to 35 unless an item says otherwise: a throwaway GitHub repo
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
   because `gh` cannot find it, with no file written in either case. Rerun when input
   parsing changes.
3. **Pull request reference is rejected before setup.** Setup: a repo with an open PR
   numbered 2. Command: `/ccl:run #2` and `/ccl:run <PR URL>`. Expected: both rejected
   before Step 0 with the reason that pull requests are not an input, and no file
   written. Rerun when input parsing changes.

## M1: low tier, no Codex

4. **Happy path.** Command: `/ccl:run #1 --no-codex`. Expected: terminal state `done`, a
   PR whose body has `Closes #1`, a comment on issue 1, and
   `.ccl/<run-id>/report.md` naming the tier as low with a reason. Rerun after any
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

## M2: medium tier

Setup for items 22 to 28: a throwaway repo with two issues (#1 and #2) in one area,
codex-lite 0.7.0 or later installed and enabled, and `codex` on PATH, unless the item
says otherwise.

22. **Bundled issues at medium tier.** Command: `/ccl:run #1 #2`. Expected: `done`, at
    least one Step 3 round with the thread id recorded in `run.md`, rejected findings
    listed with reasons, a PR body with a closing reference for each issue, and the tier
    medium, not high, because bundling alone does not raise it. Rerun after any change
    to Steps 3 to 5 or the tier rules.
23. **Parallel and ordered slices.** Setup: one plan with two independent slices, and
    one whose order of work makes the second depend on the first. Command:
    `/ccl:run #1 #2` for each. Expected: the independent slices run in one Workflow, and
    the dependent slices run in sequence, one Agent call each. Rerun after any change to
    Step 4.2.
24. **No Codex.** Command: `/ccl:run #1 #2 --no-codex`. Expected: the same path with
    Opus subagents, and the report names both swaps. Rerun after any change to the
    fallback table.
25. **Codex missing from PATH.** Setup: codex-lite installed, `codex` removed from PATH.
    Command: `/ccl:run #1 #2`. Expected: the report names the swap and the reason. Rerun
    after any change to Step 0.6.
26. **Undecidable objection.** Setup: an issue whose plan draws a blocking objection the
    orchestrator cannot decide. Command: `/ccl:run #1`. Expected: `stopped` with both
    positions printed in the report. Rerun after any change to Step 3.5.
27. **New file in a Codex review.** Setup: a slice that only adds a new file. Command:
    `/ccl:run #1 --effort medium`. Expected: a Step 5 finding that names the new file.
    This confirms Codex reviews files marked with `git add -N`, not only that the
    pre-check passes. If it does not, stop and revisit how new files are compared. Rerun
    after any codex-lite upgrade and after any change to the `git add -N` step.
28. **Codex timeout.** Setup: `.ccl.json` with `{"timeouts": {"codex": 1}}` and a large
    plan. Command: `/ccl:run #1`. Expected: the Step 3 call ends in `timeout` and the
    run ends in `blocked` naming the Codex budget. Rerun after any change to budgets or
    the status handling.

## M3: high tier

Setup for items 29 to 33: a throwaway repo with a migration file, and Codex installed
unless the item says otherwise.

29. **Risk floor applied.** Command: `/ccl:run "add a column" --effort low`. Expected:
    the run is high tier, and the report says the floor was applied and why. Rerun after
    any change to the risk floor.
30. **Risk floor not applied.** Command:
    `/ccl:run "fix a typo in the migrations README"`. Expected: low tier, and the report
    says why the floor did not apply. Rerun after any change to the risk floor.
31. **Re-evaluation after Step 4.** Setup: a task estimated medium whose implementation
    ends up removing an auth check. Command: `/ccl:run #1`. Expected: the tier rises to
    high after Step 4, and the report shows the `gpt-6-astra` final review. Rerun after
    any change to the re-evaluation rule.
32. **Round review with two sources.** Setup: a high tier run. Expected: the report
    shows Step 4.4 rounds with findings from both the orchestrator and Codex
    `gpt-6-sol`. Rerun after any change to Step 4.4.
33. **Fable fallback.** Command: a high tier run with `--no-codex`. Expected: the Step 5
    reviewer is a Fable subagent, or Opus with the Fable error recorded. This confirms
    the Agent tool accepts the Fable model override in this session and that an error
    falls through to Opus. Rerun after any change to the fallback table or the session's
    model set.

## Environment checks

34. **Codex model ids.** Setup: a ChatGPT account with Codex. Command: run a
    `/codex-lite:ask --model gpt-6-sol --timeout 60` call, then the same with
    `gpt-6-astra`. Expected: both return status `ok`. Rerun before each release and
    whenever a Codex call fails with a model error.
35. **Pre-approval of Codex calls.** Setup: default permission mode, and a command with
    `allowed-tools` limited to read-only `git` and `gh`. Command: `/ccl:run #1` at
    medium tier. Expected: record whether the codex-lite Bash call prompts. The default
    assumption is that it does. Rerun after any change to a command's `allowed-tools` or
    a Claude Code upgrade.

## Record of runs

No hand runs are recorded yet.
