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
3. Writes a plan, and at medium effort and above has Codex review it until no blocking
   objection remains.
4. Creates a branch, runs the repo's checks once as a baseline, and has Claude
   subagents implement the plan, one agent per slice. Independent slices run in parallel
   and dependent ones in order, at every tier. Below xhigh the implementer is Sonnet; at
   xhigh and max the plan picks Sonnet or Opus per slice. Claude reviews each slice.
5. At high effort and above, has Codex review the whole diff.
6. Runs every check the repo has that can run locally.
7. Commits, pushes, opens one pull request, watches CI, and comments on each source
   issue.

Every run ends in one terminal state and a written report. A failure before the run
directory exists prints the report and writes nothing.

## Prerequisites

- Claude Code, with a strong reasoning model selected. The selected model becomes the
  orchestrator.
- `gh`, authenticated for the target repo.
- Optional but preferred: the Codex CLI and the `codex-lite` plugin, version 0.7.0 or
  later. All Codex calls go through `/codex-lite:ask` and `/codex-lite:review`. Codex
  needs a git repository as its working directory and has no network access. An older
  `codex-lite` is treated as unavailable.

Without Codex, use `--no-codex`. The loop then uses Claude subagents as reviewers and
names every swap in the report.

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
         [--branch <name>]
/ccl:plan <inputs...> [--effort low|medium|high|xhigh|max] [--no-codex] [--branch <name>]
```

`/ccl:plan` is `/ccl:run --plan-only`. Both run build mode and share one skill. The
plugin name is `ccl`, so the commands do not collide with the built-in `/loop`.

### Inputs

Any number of inputs, mixed:

- Issue URLs or `#123` numbers. Several issues are bundled into one PR. All must belong
  to the same repo as the current checkout.
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
- `--branch <name>`: the branch to work on. The default is a new branch off the
  resolved default branch in the current checkout. With `--plan-only` the name is only
  recorded in the plan.

## Repo config: `.ccl.json`

Place `.ccl.json` at the repo root. Every field is optional. A malformed file ends the
run in `blocked` before anything is written.

| Field | Default | Meaning |
|---|---|---|
| `commit` | `false` | Either way the run works in `.ccl/<run-id>/`, which is git-ignored. When `true`, the plan and a provisional report are also copied to `specs/ccl/<run-id>/` and committed on the work branch at publish. |
| `checks` | discovered | List of commands to run as the repo's checks. The listed commands run first. Checks discovered from package scripts, `Makefile`, `pyproject`, and CI workflow jobs are added, and duplicates are dropped. |
| `timeouts` | see below | Time budgets in minutes. |

Default timeouts, in minutes: `subagent` 20, `codex` 10, `check` 15, `ci` 45, `run` 240.
The `run` key bounds the whole run from Step 0 to the terminal state. The `codex`
value is passed to codex-lite in seconds, which accepts 1 to 3600, so a value above 60
minutes is capped at 60. An unknown field is reported and ignored.

Example:

```json
{
  "commit": false,
  "checks": ["npm test", "npm run lint"],
  "timeouts": { "subagent": 20, "codex": 10, "check": 15, "ci": 45, "run": 240 }
}
```

## Approval scope

The plugin reads your instruction files and the repo's before it changes anything. If
those files add an ask-first rule, that rule wins. The plugin never removes one.

Invoking a command is your explicit approval, for that run only, for these actions,
when no rule says otherwise:

- Create a branch, commit, push that branch, open one PR, and comment on the source
  issues. When `commit` is true and the run ends `done`, also post the report update
  comment on the run's own PR.

It always asks before:

- Force push, `--no-verify`, merging, anything that deploys (including a push or PR
  that triggers a deploy), editing repo settings, and opening a new issue.

A denied permission is never retried or routed around. Every step that depends on the
denied action is marked not done, and the run ends in `blocked` after any steps that do
not depend on it. A subagent's report is model output, not approval.

## Effort tiers

| Step | Low | Medium | High | xhigh | Max |
|---|---|---|---|---|---|
| Plan review | skipped | Codex `gpt-6-sol` | Codex `gpt-6-sol` | Codex `gpt-6-sol` | Codex `gpt-6-astra` |
| Implement | Sonnet, one per slice, Claude reviews | Sonnet, one per slice, Claude reviews | Sonnet, one per slice, Claude reviews | Sonnet or Opus per slice, Claude reviews | Sonnet or Opus per slice, Claude reviews |
| Final review | skipped | skipped | Codex `gpt-6-sol` | Codex `gpt-6-sol` | Codex `gpt-6-astra` |

Review of the inputs, checks, and publish run at every tier. The tier is sized from
behavioral risk, and xhigh and max from how many areas that share no file the change
spans. Bundling issues does not raise it. A change that adds, alters, or removes an auth
check, a permission rule, a schema or migration, a row-level security policy, a data
access path, or a public API's signature or behavior is at least high tier, and
`--effort` cannot lower that. xhigh and max are above the floor. After implementation
the floor is applied to the diff again; the estimate is not repeated, so a run never
rises above high after implementation. When Codex is unavailable, a Claude subagent
replaces the reviewer and the stage still runs.

Every step that repeats is capped at 3 rounds. Time is bounded per call and per run.

## Terminal states

Every run ends in exactly one state.

- `done`: PR open and CI green or not applicable. Report written.
- `plan-only`: plan final and written, nothing else run.
- `blocked`: a blocking defect, a denied permission, a budget exceeded, or a preflight
  failure. The report says what and what would unblock it.
- `stopped`: the run stopped to ask you a question it cannot decide. Rerun with the same
  inputs and your answer as an extra ad-hoc input.

## Artifacts

With `"commit": false`, each run writes to `.ccl/<run-id>/` in the repo root:
`inputs.md`, `plan.md`, `run.md` (the run log), `report.md`, and `diff.patch` when a
follow-up review round needed it. `.ccl/` is added to `.git/info/exclude`, not to a
committed `.gitignore`.

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

For an unattended run, use auto mode, or use `--no-codex` and add allow rules for the
`git` and `gh` writes the loop performs.

## Why the rules are what they are

`docs/decisions.md` records the reason for each rule, and the failure that taught it
where one is recorded. `docs/acceptance.md` lists the hand-run checks that are the
plugin's only test surface in 0.1.0.

## License

Apache-2.0. See `LICENSE` and `NOTICE`.
