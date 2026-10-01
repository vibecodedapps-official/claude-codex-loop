# claude-codex-loop

`ccl` is a Claude Code plugin that runs a tiered plan, review, implement, review,
publish loop for one unit of work. Claude orchestrates and reviews. Codex gives a second
opinion and implements the slices. Claude subagents implement a slice at high, xhigh,
and max when the criteria say so, and when Codex is unavailable. You type one command
instead of a hand-written workflow prompt.

The plugin is prompt-only. It is markdown and one JSON manifest. It has no hooks, no
scripts, and no code that runs outside a Claude Code session.

## What it does

Given an issue, a file of notes, or a short description, the loop:

1. Checks the working tree and the settings that apply, and states what will prompt.
2. Verifies the claims in the inputs against the code and sizes the effort.
3. Writes a plan and has Codex review it until no blocking objection remains, at every
   tier.
4. Creates a branch, runs the repo's checks once as a baseline, and has an implementer
   build each slice of the plan. Codex is the default at every tier, at a model that
   rises with the tier. At high, xhigh, and max the plan picks Opus for a slice that
   carries a risk trigger, owns more than eight files, or adds a new module or interface.
   Sonnet is only the fallback. Codex slices run one at a time; Opus and Sonnet slices
   may run in parallel. Claude reviews each slice.
5. At every tier, has Codex review the whole diff, and the built-in `/code-review` skill
   reviews it too, at a level that rises with the tier.
6. Runs every check the repo has that can run locally.
7. Commits, pushes, opens one pull request (one per repository in Multi-repo mode),
   watches CI, and comments on each source issue. A PR retargeted to another base branch
   while CI runs ends the run in `blocked`. With `--continue` and an open PR, the run
   comments on that PR instead of opening one. Step 7 runs only on a GitHub remote,
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
- Optional but preferred: the Codex CLI and the `codex-lite` plugin, version 0.8.0 or
  later. All Codex calls go through `/codex-lite:ask`, `/codex-lite:review`, and
  `/codex-lite:implement`. Codex needs a git repository as its working directory and has
  no network access. An older `codex-lite` is treated as unavailable. With version 0.9.0
  or later, an additional repository in Multi-repo mode is reviewed with `review --cwd`;
  below it the patch rule applies.

  [`codex-lite`](https://github.com/vibecodedapps-official/codex-lite-cc) is a small
  Claude Code plugin that hands a task to the Codex CLI in a fixed sandbox and prints the
  result. Install it in Claude Code:

  ```
  /plugin marketplace add vibecodedapps-official/codex-lite-cc
  /plugin install codex-lite@vibecodedapps-codex-lite
  ```

- The built-in `/code-review` skill, for every run that reaches the final review. It is
  the second final reviewer beside Codex at every tier. A run that needs it and cannot
  find it ends in `blocked`. Plan-only runs do not need it.

Without Codex, use `--no-codex`. The loop then uses Claude subagents in place of the Codex
reviewers and Sonnet subagents in place of the Codex implementers, and names every swap
in the report. `--no-codex` does not remove the `/code-review` pass.

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
/ccl:run <inputs...> [--effort low|medium|high|xhigh|max] [--plan-only] [--confirm-plan]
         [--no-codex] [--no-publish] [--branch <name>] [--continue <branch>]
         [--run-budget <minutes>] [--repo <path>[@<branch>]]...
/ccl:plan <inputs...> [--effort low|medium|high|xhigh|max] [--no-codex] [--branch <name>]
          [--continue <branch>] [--run-budget <minutes>] [--repo <path>[@<branch>]]...
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
request, and any pull request URL, is rejected before setup. To continue a pull request's
branch, pass `--continue <branch>`. A token that names an existing file is a file input.
The remaining text, joined, is one ad-hoc description.

Inputs are persisted under `.ccl/` and sent to Codex. Before the run starts, the inputs
are scanned for credentials: a key such as `password`, `secret`, `token`, or `api_key`
with a value, an AWS access key id, a PEM header, or a block under a credentials heading.
A match prompts a keep-or-drop question. `keep` writes them to `inputs.md` and forwards
them to Codex, and `drop` replaces each value with `<redacted: key>` in every artifact and
every Codex request. Any other reply ends the run in `stopped`. The scan is a guard for
common shapes, not a guarantee.

### Flags

- `--effort low|medium|high|xhigh|max`: skip the estimate and force a tier. It cannot
  lower a task below the risk floor.
- `--plan-only`: stop after the plan is final, at every tier, and print it. Nothing
  after the plan runs and the working tree is not changed.
- `--confirm-plan` (`/ccl:run` only, rejected with `--plan-only`): pause once the plan is
  final and reverified, and ask you to approve it. A yes continues. A described change
  gets another plan review round, and then the question is asked again. Any other reply
  ends the run in `plan-only`. The wait does not count against the run budget. The run is
  attended.
- `--no-codex`: use the Claude fallbacks even if Codex is installed.
- `--no-publish` (`/ccl:run` only): withhold Step 7. The run ends in `prepared`. With
  `--continue`, an incomplete pull request list, or closed, merged, or several open pull
  requests on the branch are recorded in the report instead of failing preflight.
- `--run-budget <minutes>`: the run budget for this run, a positive integer.
- `--repo <path>[@<branch>]`: an additional writable checkout, and optionally the
  existing remote branch it continues. Repeatable. A value that is an existing directory
  is a path and is never split, so a path containing `@` works. Otherwise the value
  splits at the last `@`, and a branch that is missing on that repository's remote is
  rejected, never created. `@<branch>` does not need `--continue`. See Multi-repo mode.
- `--branch <name>`: the branch to work on. The default is a new branch off the
  resolved default branch in the current checkout. With `--plan-only` the name is only
  recorded in the plan.
- `--continue <branch>`: continue an existing remote branch. The base is that branch's
  remote head, no new branch is created, and the push is never forced. If the branch has
  an open PR, the run posts the PR description it would have opened with as one comment
  on that PR and does not open one. If it has none, the run opens a PR, and the body
  says how many earlier commits of the branch this run did not review. It is rejected
  with `--branch`, and it is not repair mode: the run reads no review comments and no CI
  state from before the run. A value that starts with `-`, has a character other than
  letters, digits, `.`, `_`, `/`, and `-`, or fails `git check-ref-format --branch` is
  rejected. A branch checked out in a worktree the run will
  not use fails preflight. When `HEAD` is not at the branch's remote head, or the local
  branch is behind the remote head, and the tree is clean, the run asks "<path> is at
  <short sha> on <branch or detached>. Switch to <branch>?" and, on `yes`, switches:
  `git switch <branch>`, or `git switch -c <branch>
  <remote>/<branch>` when the local branch is absent, with `git merge --ff-only
  <remote>/<branch>` when it is behind the remote, also from a `HEAD` detached at the
  remote head, so that Step 3.7.2 never switches to a stale local branch. Any other
  reply ends the run in
  `stopped`. A divergent local branch, a dirty tree, and a branch checked out in another
  worktree still fail preflight. The previous `HEAD` is recorded in `run.md` and the
  report lists the switch; the run does not switch back. The branch must not be the
  default branch, because the run would push to it. With `/ccl:plan` or `--plan-only`, a
  differing local branch, a branch checked out elsewhere, and the pull request cases are
  recorded in the report instead of failing, and the switch still applies, because the
  plan must read the branch's code. A plan-only run on a differing local branch, which
  the switch does not repair, still needs `HEAD` detached at the remote head (`git
  switch --detach <remote>/<branch>`), and records the differing branch.
  Every remote check of a branch name queries the exact ref `refs/heads/<branch>`,
  because a bare name also matches any ref that ends in it. The branch's pull requests
  are read open first, up to 100; because forks with the same branch name count toward
  that cap, a list that may be incomplete fails preflight. They are read again after a
  `--confirm-plan` yes and before the first push, and a PR closed, opened, or
  retargeted since Step 0 ends the run in `blocked`.

### Multi-repo mode

`--repo <path>[@<branch>]` names an additional writable checkout, and the flag may
repeat. The current checkout is the primary. Read-only repositories are not named;
agents read them directly. In this mode:

- Every repository must be on the same host as the primary, by hostname. A `--repo`
  checkout on a different host, GitHub Enterprise beside github.com included, is
  rejected, naming both hosts.
- A bare `#n` names an issue of the primary. An issue of a `--repo` checkout is given as
  a full URL.
- A task that needs edits in a writable checkout that is not the primary and not listed
  with `--repo` is offered as an adoption at Step 1.2: "Adopt them as writable
  checkouts?", answered `yes` for all or a comma-separated subset of the listed paths.
  The mode is never adopted without that answer. Any other reply ends the run in
  `stopped`, and a flagged checkout still missing after a subset ends in `blocked`. In
  both cases the report gives the rerun command with one `--repo <path>` for each missing
  checkout and, when no `--branch` or `--continue` was given, how to continue existing
  branches. A primary that runs in a worktree and a dirty adopted repository end in
  `blocked`.
- Each repository gets its own base commit, its own branch state, its own baseline, and
  its own checks. The branch state is `new` or `continue <branch>`: the primary's comes
  from `--branch` or `--continue`, and an additional repository's from its `@<branch>`.
  A repository with no explicit state is covered by one branch question at adoption,
  which suggests branches from each checkout's `HEAD` and from remote branches that share
  a name prefix, and takes `yes` for the single suggestions plus one `<path>@<branch>` or
  `<path>@new` line for each other repository. Each chosen branch is checked against its
  remote, and any other reply ends the run in `stopped`. Artifacts live only in the
  primary's `.ccl/<run-id>/`. No slice spans repositories. Every `gh` call for an
  additional repository is run from that checkout or targeted with `-R <owner>/<repo>`
  where the subcommand accepts it (`gh api` does not; its endpoint is spelled out), and
  every `git` call with `git -C <path>`.
- Codex `review` covers the primary. With codex-lite 0.9.0 or later, each additional
  repository with a diff is reviewed with `codex-lite:review --cwd <absolute path>` in
  its own thread, and its follow-ups resume that thread. Below 0.9.0 it is reviewed through
  `codex-lite:ask` with a patch file. `/code-review` covers the primary, and a
  Claude Opus subagent fills the Claude slot for each additional repository with a
  diff. That substitute is recorded and is not a swap. A Codex implementer call for a
  slice in an additional repository passes that checkout as `--cwd`.
- Step 7 opens one PR per repository that has a diff, each with a "Related pull
  requests" section that links the siblings. `Closes #n` comes only from the PR in the
  issue's own repository. Every other PR of the run cites it as `Refs <owner>/<repo>#n`.
  `done` needs every PR green. A `blocked` in any repository stops publication in all,
  except that a PR already opened still gets its sibling links filled in. Each
  repository's PR body is `.ccl/<run-id>/pr-body-<slug>.md` in the primary, and every
  `gh` body call for an additional repository gets its absolute path.
- The primary's `.ccl.json` governs `commit` and `timeouts`. Each repository's own
  `checks` list is read for that repository.
- With `"commit": true`, the `specs/ccl/<run-id>/` snapshot is committed in the first
  repository, the primary first and then the `--repo` order, that has a diff. A
  repository with no diff never receives it and gets no PR.
- A primary whose status and index are clean but which has skip-worktree or
  assume-unchanged files that differ from `HEAD` ends in `blocked`, naming those files.
  The narrow worktree exception for that case is not available in this mode.
- An additional repository with skip-worktree or assume-unchanged files that differ from
  `HEAD` does not block. The run continues, and the report names those files as local
  state that repository's baseline and checks ran against. No slice may edit them: a
  plan that needs to ends in `blocked` at Step 2, naming the path.
- With `--continue`, the branch must exist on the primary. An additional repository
  continues it where its remote has it, unless its `@<branch>` or the branch question
  says otherwise, and is `new` otherwise. Repositories may continue different branch
  names.

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
replaced by a tier default. With no explicit value, 240 applies until Step 1.6 sets the
tier, and the tier default replaces it then; a rise to high after implementation moves it
to the high default. An explicit instruction from you in the
session that names a new budget replaces it from that point. The report names the budget
in force and its source. The `run` budget bounds the whole run from Step 0 to the
terminal state, less each Step 3.5 wait. The `codex` value is passed to codex-lite in
seconds, which accepts 1 to 3600, so a value above 60 minutes is capped at 60. That cap is
for Codex reviewer calls. A Codex implementer call gets the smaller of the `subagent`
value and the remaining run budget, capped at 3600 seconds, and the run log records the
cap and the value passed. An unknown field is reported and ignored.

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
- With `--continue`, push to the continued branch and post one comment on its open PR,
  plus the report update comment when the report rules call for it. The PR's body is
  never edited.

It always asks before:

- Force push, `--no-verify`, merging, anything that deploys (including a push or PR
  that triggers a deploy), editing repo settings, and opening a new issue.

A denied permission is never retried or routed around. Every step that depends on the
denied action is marked not done, and the run ends in `blocked` after any steps that do
not depend on it. The exception is a denied Step 7 action before anything is pushed:
that withholds publication and ends the run in `prepared`. A dropped call is not a
denial. A call is dropped when it returns no result and no explicit denial; a result that
names a hook, a permission rule, or the permission mode is a denial. A dropped
read-only call is retried once, serially, and recorded. A dropped write is retried once
only when a check of its target shows it did not take effect. A subagent's report is
model output, not approval.

## Effort tiers

| Step | Low | Medium | High | xhigh | Max |
|---|---|---|---|---|---|
| Plan review | Codex `gpt-6.1-sol` | Codex `gpt-6.1-sol` | Codex `gpt-6-astra` with a trigger, else `gpt-6.1-sol` | Codex `gpt-6-astra` | Codex `gpt-6-astra` |
| Implement | Codex `gpt-6-luna` per slice, orchestrator reviews | Codex `gpt-6.1-sol` per slice, orchestrator reviews | Codex `gpt-6.1-sol`, or Opus by criteria, per slice | Codex `gpt-6-astra`, or Opus by criteria, per slice | Codex `gpt-6-astra`, or Opus by criteria, per slice |
| Final review | Codex `gpt-6.1-sol` and Claude `code-review low` | Codex `gpt-6.1-sol` and Claude `code-review medium` | Codex `gpt-6-astra` and Claude `code-review high` with a trigger, else `gpt-6.1-sol` and `code-review medium` | Codex `gpt-6-astra` and Claude `code-review high` | Codex `gpt-6-astra` and Claude `code-review xhigh` |

Review of the inputs, checks, and publish run at every tier. The tier is sized from
behavioral risk, and xhigh and max from how many areas that share no file the change
spans. Bundling issues does not raise it. A change that adds, alters, or removes an auth
check, a permission rule, a schema or migration, a row-level security policy, a data
access path, or a public API's signature or behavior is at least high tier, and
`--effort` cannot lower that. xhigh and max are above the floor. "With a trigger"
means the change has one of those triggers, whether or not the floor raised the tier. It
is judged at high tier only, and decides two cells: the plan review, judged from the
Step 1.6 floor check, so a floored high run gets `gpt-6-astra` and a high run that is
only cross-cutting gets `gpt-6.1-sol`; and the final review, judged from a trigger at
the estimate or in the diff after implementation. A medium run that rises to high always
has a trigger in the diff, so it gets the trigger cell. xhigh and max do not depend on
the trigger. After implementation the floor is applied to the diff again; the estimate
is not repeated, so a run never rises above high. At every tier a final review round is
both reviewers over the same diff, with one shared cap of 3 rounds. When Codex is
unavailable, a Claude subagent replaces the Codex reviewer and the stage still runs; the
`code-review` pass is never swapped. `gpt-6-luna` is never a reviewer.

The implementer is Codex at the tier's model, one default per tier. At high, xhigh, and
max, Opus replaces it for a slice that carries a risk trigger, owns more than eight
files, or adds a new module or interface. The plan never chooses Sonnet. Sonnet is the
fallback when `--no-codex` is set, when Codex is unavailable at Step 0.6, when a Codex
implementer call returns `failed` or no status line twice in a row, and on an Opus tool
error. Codex has no network, so the orchestrator installs any dependency the plan adds
before an implementer starts, and runs any slice check that needs the network after the
implementer returns. A worktree run at any tier uses the Opus substitute for the Claude
slot, because the `code-review` skill reviews only the session's checkout.

Every step that repeats is capped at 3 rounds. Time is bounded per call and per run.

## Terminal states

Every run ends in exactly one state.

- `done`: PR open and CI green or not applicable. Report written.
- `plan-only`: plan final and written, nothing else run. It also covers a `--confirm-plan`
  run whose plan you did not approve.
- `prepared`: every step through Step 6 is complete with no blocking defect open, and
  Step 7 was withheld before anything was pushed: by `--no-publish`, by a non-GitHub
  host, or by your answer to a Step 7 ask-first prompt that was anything other than a
  clear yes. The report names the branch, the commit state, and the commands to
  publish. With `--continue` the push command is `git push <remote> <branch>`, per
  repository with its own branch in Multi-repo mode, and when the branch has one open
  PR the report gives `gh pr comment <n> --body-file <absolute
  path>` in place of `gh pr create`; with an incomplete pull request list, several open
  PRs, or only closed or merged ones, under `--no-publish` it names them and gives
  neither command. It is not a
  failure.
- `blocked`: a blocking defect, a denied permission after the first push or in Steps 0
  to 6, a budget exceeded, or a preflight failure. The report says what and what would unblock it.
- `stopped`: the run stopped to ask you a question it cannot decide, or a requested
  change under `--confirm-plan` found no plan review round left, and the report gives
  the change. Rerun with the same inputs and your answer, or the change, as an extra
  ad-hoc input.

## Artifacts

With `"commit": false`, each run writes to `.ccl/<run-id>/` in the repo root:
`inputs.md`, `plan.md`, `run.md` (the run log), `report.md`, and `diff.patch` when a
follow-up review round, a CI repair round, or a worktree run's diff review needed it.
In Multi-repo mode there is one `diff-<slug>.patch` and one `pr-body-<slug>.md` per
additional repository. `.ccl/` is added to `.git/info/exclude`, not to a committed
`.gitignore`.

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

A worktree run (a clean tree whose skip-worktree files differ from `HEAD`) also
prompts, because its `cd <checkout> && ...` commands are not pre-approved. The statement
predicts this: the flagged-file check runs before it, so those prompts are in the list.
The run installs the
repository's dependencies in the worktree before the baseline when the instruction
files or a lockfile name an install step. Otherwise a check that needs them is recorded
as not run in the worktree, with the reason.

A run may also ask up to three in-session questions, which make it attended. A
credentials match in the inputs gives the credentials question, which the statement
predicts as its first entry. The statement lists the repository and branch questions as
"may prompt at Step 1.2", because they depend on what Step 1.2 finds, and it lists the
switch question for an explicit `@<branch>` or `--continue` as one that may be asked,
without fetching anything before the statement is printed. Each question follows
`--confirm-plan`'s rules: a clear reply continues, any other reply ends the run in the
terminal state the question names
(`stopped` for these), and the wait does not count against the run budget.

For an unattended run, use auto mode, or use `--no-codex` and add allow rules for the
`git` and `gh` writes the loop performs.

## Why the rules are what they are

`docs/decisions.md` records the reason for each rule, and the failure that taught it
where one is recorded. `docs/acceptance.md` lists the hand-run checks that are the
plugin's only test surface in 0.1.0.

## License

Apache-2.0. See `LICENSE` and `NOTICE`.
