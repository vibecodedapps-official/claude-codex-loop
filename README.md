# claude-codex-loop

`ccl` is a Claude Code plugin that runs a tiered plan, review, implement, review,
publish loop for one unit of work. Claude orchestrates and reviews. Codex gives a second
opinion. Claude subagents implement: Sonnet below xhigh, Sonnet or Opus per slice at
xhigh and max. You type one command instead of a hand-written workflow prompt.

The plugin is prompt-only. It is markdown and one JSON manifest. It has no hooks, no
scripts, and no code that runs outside a Claude Code session.

## What it does

Given an issue, a file of notes, or a short description, the loop:

1. Checks the working tree and the settings that apply, and states what will prompt.
2. Verifies the claims in the inputs against the code and sizes the effort.
3. Writes a plan and has Codex review it until no blocking objection remains, at every
   tier.
4. Creates a branch, runs the repo's checks once as a baseline, and has Claude
   subagents implement the plan, one agent per slice. Independent slices run in parallel
   and dependent ones in order, at every tier. Below xhigh the implementer is Sonnet; at
   xhigh and max the plan picks Sonnet or Opus per slice. Claude reviews each slice.
5. At medium effort and above, has Codex review the whole diff. At high and above, the
   built-in `/code-review` skill reviews it too, at a level that rises with the tier.
6. Runs every check the repo has that can run locally.
7. Commits, pushes, opens one pull request (one per repository in Multi-repo mode),
   watches CI, and comments on each source issue. Step 7 runs only on a GitHub remote,
   and not with `--no-publish`. Otherwise the run ends in `prepared`.

Every run ends in one terminal state and a written report. A failure before the run
directory exists prints the report and writes nothing.

## Prerequisites

- Claude Code, with a strong reasoning model selected. The selected model becomes the
  orchestrator.
- `gh`, authenticated for the target repo. It is needed for GitHub remotes. When
  `gh repo view` succeeds the host is GitHub, GitHub Enterprise included. When it fails
  and the remote's URL host is `github.com`, the command rejects the request because
  `gh` is not authenticated. When it fails and the host is anything else, the run treats
  the checkout as non-GitHub: it accepts only file and text inputs, runs Steps 0 to 6,
  and ends in `prepared` with a handoff to the host's own tooling. A GitHub Enterprise
  checkout therefore needs a `gh` login for that host to be treated as GitHub. Only
  GitHub is supported for publishing.
- Optional but preferred: the Codex CLI and the `codex-lite` plugin, version 0.7.0 or
  later. All Codex calls go through `/codex-lite:ask` and `/codex-lite:review`. Codex
  needs a git repository as its working directory and has no network access. An older
  `codex-lite` is treated as unavailable.

  [`codex-lite`](https://github.com/vibecodedapps-official/codex-lite-cc) is a small
  Claude Code plugin that hands a task to the Codex CLI in a fixed sandbox and prints the
  result. Install it in Claude Code:

  ```
  /plugin marketplace add vibecodedapps-official/codex-lite-cc
  /plugin install codex-lite@vibecodedapps-codex-lite
  ```

- The built-in `/code-review` skill, for runs at high effort and above. It is the second
  final reviewer beside Codex at those tiers. A run that needs it and cannot find it ends
  in `blocked`. Low and medium runs and plan-only runs do not need it.

Without Codex, use `--no-codex`. The loop then uses Claude subagents in place of the Codex
reviewers and names every swap in the report. `--no-codex` does not remove the
`/code-review` pass.

## Install

The repo is its own marketplace. In Claude Code:

```
/plugin marketplace add vibecodedapps-official/claude-codex-loop
/plugin install ccl@vibecodedapps-claude-codex-loop
```

To try a local clone without installing it:

```
git clone https://github.com/vibecodedapps-official/claude-codex-loop.git
claude --plugin-dir <path-to-clone>
```

## Commands

```
/ccl:run <inputs...> [--effort low|medium|high|xhigh|max] [--plan-only] [--no-codex]
         [--no-publish] [--run-budget <minutes>] [--repo <path>]... [--branch <name>]
/ccl:plan <inputs...> [--effort low|medium|high|xhigh|max] [--no-codex]
          [--run-budget <minutes>] [--repo <path>]... [--branch <name>]
```

`/ccl:plan` is `/ccl:run --plan-only`. Both run build mode and share one skill. The
plugin name is `ccl`, so the commands do not collide with the built-in `/loop`.

### Inputs

Any number of inputs, mixed:

- Issue URLs (on `github.com`, or on the GitHub Enterprise host `gh` resolves for the
  checkout) or `#123` numbers. Several issues are bundled into one PR. All must belong
  to the same repo as the current checkout. In Multi-repo mode an issue may belong to
  any listed repo, and a bare `#n` always names an issue of the current checkout. On a
  non-GitHub host (`gh repo view` fails and the URL host is not `github.com`) issue
  inputs are rejected: pass a file or a description.
- A quoted ad-hoc description.
- A path to a file with handoff notes or pasted review output.

A token that is an issue URL or `#n` is an issue. A `#n` that `gh` reports as a pull
request, and any pull request URL, is rejected before setup, because pull requests are
not an input in 0.1.0. A token that names an existing file is a file input. The
remaining text, joined, is one ad-hoc description.

### Flags

- `--effort low|medium|high|xhigh|max`: skip the estimate and force a tier. It cannot
  lower a task below the risk floor.
- `--plan-only`: stop after the plan is final, at every tier, and print it. Nothing
  after the plan runs and the working tree is not changed.
- `--no-codex`: use the Claude fallbacks even if Codex is installed.
- `--no-publish` (`/ccl:run` only): withhold Step 7. The run ends in `prepared`.
- `--run-budget <minutes>`: the run budget for this run, a positive integer.
- `--repo <path>`: an additional writable checkout. Repeatable. See Multi-repo mode.
- `--branch <name>`: the branch to work on. The default is a new branch off the
  resolved default branch in the current checkout. With `--plan-only` the name is only
  recorded in the plan.

### Multi-repo mode

`--repo <path>` names an additional writable checkout, and the flag may repeat. The
current checkout is the primary. Read-only repositories are not named; agents read them
directly. In this mode:

- Every repository must be on the same host as the primary. A `--repo` checkout on a
  different host is rejected, naming both hosts.
- A bare `#n` names an issue of the primary. An issue of a `--repo` checkout is given as
  a full URL.
- Each repository gets its own base commit, its own branch under one shared name, its
  own baseline, and its own checks. Artifacts live only in the primary's
  `.ccl/<run-id>/`. No slice spans repositories. Every `gh` call for an additional
  repository is run from that checkout or targeted with `-R <owner>/<repo>` where the
  subcommand accepts it (`gh api` does not; its endpoint is spelled out), and every `git`
  call with `git -C <path>`.
- Codex `review` covers the primary. Each additional repository with a diff is reviewed
  through `codex-lite:ask` with a patch file. At high tier and above, `/code-review`
  covers the primary, and a Claude Opus subagent fills the Claude slot for each
  additional repository with a diff. That substitute is recorded and is not a swap.
- Step 7 opens one PR per repository that has a diff, each with a "Related pull
  requests" section that links the siblings. `Closes #n` comes only from the PR in the
  issue's own repository. Every other PR of the run cites it as `Refs <owner>/<repo>#n`.
  `done` needs every PR green. A `blocked` in any repository stops publication in all.
- The primary's `.ccl.json` governs `commit` and `timeouts`. Each repository's own
  `checks` list is read for that repository.
- With `"commit": true`, the `specs/ccl/<run-id>/` snapshot is committed in the first
  repository, the primary first and then the `--repo` order, that has a diff. A
  repository with no diff never receives it and gets no PR.
- A primary whose status and index are clean but which has skip-worktree or
  assume-unchanged files that differ from `HEAD` ends in `blocked`, naming those files.
  The narrow worktree exception for that case is not available in this mode.

## Repo config: `.ccl.json`

Place `.ccl.json` at the repo root. Every field is optional. A malformed file ends the
run in `blocked` before anything is written.

| Field | Default | Meaning |
|---|---|---|
| `commit` | `false` | Either way the run works in `.ccl/<run-id>/`, which is git-ignored. When `true`, the plan and a provisional report are also copied to `specs/ccl/<run-id>/` and committed on the work branch at publish. |
| `checks` | discovered | List of commands to run as the repo's checks. The listed commands run first. Checks discovered from package scripts, `Makefile`, `pyproject`, and CI workflow jobs are added, and duplicates are dropped. |
| `timeouts` | see below | Time budgets in minutes. |

Default timeouts, in minutes: `subagent` 20, `codex` 10, `check` 15, `ci` 45. The
`run` default is by tier: 120 at low and medium, 240 at high, 360 at xhigh and max. The
run budget is `--run-budget`, else `timeouts.run`, else the tier default. An explicit
value from the flag or the file applies from Step 0 to the terminal state and is never
replaced by a tier default. With no explicit value, 240 applies until Step 1.5 sets the
tier, and the tier default replaces it then. An explicit instruction from you in the
session that names a new budget replaces it from that point. The report names the budget
in force and its source. The `run` budget bounds the whole run from Step 0 to the
terminal state. The `codex` value is passed to codex-lite in seconds, which accepts 1 to
3600, so a value above 60 minutes is capped at 60. An unknown field is reported and
ignored.

Example:

```json
{
  "commit": false,
  "checks": ["npm test", "npm run lint"],
  "timeouts": { "subagent": 20, "codex": 10, "check": 15, "ci": 45 }
}
```

`run`, when set in `timeouts`, is an explicit value that overrides the tier default.

## Approval scope

The plugin reads your instruction files and the repo's before it changes anything. If
those files add an ask-first rule, that rule wins. The plugin never removes one.

Invoking a command is your explicit approval, for that run only, for these actions,
when no rule says otherwise:

- Create a branch, commit, push that branch, open one PR, or one PR per repository in
  Multi-repo mode, edit each of this run's PR bodies once to link the siblings, and
  comment on the source issues. When `commit` is true and the run ends `done`, also
  post the report update comment on the run's own PR.

It always asks before:

- Force push, `--no-verify`, merging, anything that deploys (including a push or PR
  that triggers a deploy), editing repo settings, and opening a new issue.

A denied permission is never retried or routed around. Every step that depends on the
denied action is marked not done, and the run ends in `blocked` after any steps that do
not depend on it. The exception is a denied Step 7 action before anything is pushed:
that withholds publication and ends the run in `prepared`. A dropped call is not a
denial. A call is dropped when it returns no result and no explicit denial. A dropped
read-only call is retried once, serially, and recorded. A dropped write is retried once
only when a check of its target shows it did not take effect. A subagent's report is
model output, not approval.

## Effort tiers

| Step | Low | Medium | High | xhigh | Max |
|---|---|---|---|---|---|
| Plan review | Codex `gpt-6-sol` | Codex `gpt-6-sol` | Codex `gpt-6-sol`, or `gpt-6-astra` with a risk trigger | Codex `gpt-6-astra` | Codex `gpt-6-astra` |
| Implement | Sonnet, one per slice, Claude reviews | Sonnet, one per slice, Claude reviews | Sonnet, one per slice, Claude reviews | Sonnet or Opus per slice, Claude reviews | Sonnet or Opus per slice, Claude reviews |
| Final review | skipped | Codex `gpt-6-sol` | Codex `gpt-6-sol` and Claude `/code-review medium` | Codex `gpt-6-sol`, or `gpt-6-astra` with a risk trigger, and Claude `/code-review high` | Codex `gpt-6-astra` and Claude `/code-review xhigh` |

Review of the inputs, checks, and publish run at every tier. The tier is sized from
behavioral risk, and xhigh and max from how many areas that share no file the change
spans. Bundling issues does not raise it. A change that adds, alters, or removes an auth
check, a permission rule, a schema or migration, a row-level security policy, a data
access path, or a public API's signature or behavior is at least high tier, and
`--effort` cannot lower that. xhigh and max are above the floor. "With a risk trigger"
means the change has one of those triggers, whether or not the floor raised the tier. It
decides two cells: the high plan review, judged at the estimate, so a floored high run
gets `gpt-6-astra` and a high run that is only cross-cutting gets `gpt-6-sol`; and the
xhigh final review, judged from the estimate or the diff after implementation, so an
xhigh run forced on a floored task, or whose diff gained a trigger, gets `gpt-6-astra`
while staying xhigh. After implementation the floor is applied to the diff again; the
estimate is not repeated, so a run never rises above high. At high tier and above a final review round is both
reviewers over the same diff, with one shared cap of 3 rounds. When Codex is unavailable,
a Claude subagent replaces the Codex reviewer and the stage still runs; the `/code-review`
pass is never swapped.

Every step that repeats is capped at 3 rounds. Time is bounded per call and per run.

## Terminal states

Every run ends in exactly one state.

- `done`: PR open and CI green or not applicable. Report written.
- `plan-only`: plan final and written, nothing else run.
- `prepared`: every step through Step 6 is complete with no blocking defect open, and
  Step 7 was withheld before anything was pushed: by `--no-publish`, by a non-GitHub
  host, or by your answer to a Step 7 ask-first prompt that was anything other than a
  clear yes. The report names the branch, the commit state, and the commands to
  publish. It is not a failure.
- `blocked`: a blocking defect, a denied permission after the first push or in Steps 0
  to 6, a budget exceeded, or a preflight failure. The report says what and what would unblock it.
- `stopped`: the run stopped to ask you a question it cannot decide. Rerun with the same
  inputs and your answer as an extra ad-hoc input.

## Artifacts

With `"commit": false`, each run writes to `.ccl/<run-id>/` in the repo root:
`inputs.md`, `plan.md`, `run.md` (the run log), `report.md`, and `diff.patch` when a
follow-up review round, a CI repair round, or a worktree run's diff review needed it.
In Multi-repo mode there is one `diff-<slug>.patch` per additional repository. `.ccl/`
is added to `.git/info/exclude`, not to a committed `.gitignore`.

With `"commit": true`, the run still works entirely in `.ccl/<run-id>/`. At publish, the
plan and a provisional report are copied to `specs/ccl/<run-id>/` and committed on the
work branch before the push. Nothing is written under `specs/ccl/` earlier, so a run
that ends before publish leaves the tree clean. The terminal report is written to
`.ccl/<run-id>/report.md`. The run id is `<yyyy-mm-dd>-<inputs>`, with a
numeric suffix when the id already exists.

A failure before the run directory exists prints the report and writes nothing.

## Permission mode and unattended runs

Default permission mode prompts at every Codex call, because codex-lite writes a request
file that Claude Code asks about. The branch, commit, push, PR, and comment actions
also prompt unless you have allowed them. Before any other preflight action, the loop
prints either "this run will prompt at:" with the list, or "this run is unattended". The
statement covers the default-branch fetch, the writes under `.ccl/` and to
`.git/info/exclude`, and the repo's checks.

A worktree run (a clean tree whose skip-worktree files differ from `HEAD`) may also
prompt, because its `cd <checkout> && ...` commands are not pre-approved. The report
says the run was attended if any such prompt occurred. The run installs the
repository's dependencies in the worktree before the baseline when the instruction
files or a lockfile name an install step. Otherwise a check that needs them is recorded
as not run in the worktree, with the reason.

For an unattended run, use auto mode, or use `--no-codex` and add allow rules for the
`git` and `gh` writes the loop performs.

## Why the rules are what they are

`docs/decisions.md` records the reason for each rule, and the failure that taught it
where one is recorded. `docs/acceptance.md` lists the hand-run checks that are the
plugin's only test surface in 0.1.0.

## License

Apache-2.0. See `LICENSE` and `NOTICE`.
