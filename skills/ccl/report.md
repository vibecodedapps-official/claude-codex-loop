# Final report template

Read this file at every terminal state. Fill in every section. Write "none" for an empty
one; never drop a section. Compile the report from the run log, not from memory.

## When and where

Every run ends in exactly one terminal state, writes the report, and prints it.

- Terminal states: `done` (PR open, CI green or not applicable), `plan-only` (plan final
  and written, nothing else run, or a `--confirm-plan` run the user did not approve),
  `prepared` (every step through Step 6 is complete with no
  blocking defect open, and Step 7 was withheld before anything was pushed: by
  `--no-publish`, by a non-GitHub host, or by the user answering a Step 7 ask-first prompt
  with anything other than a clear yes; not a failure), `blocked` (a blocking defect, a
  denied permission, a budget exceeded, or a preflight failure), `stopped` (the run stopped
  to ask the user a question it cannot decide, or a change requested under
  `--confirm-plan` found no plan review round left).
- A failure before Step 0.5, when the run directory does not exist yet, prints the report
  and writes nothing. The tree may be dirty and the artifacts directory may not be ignored
  yet. The printed report names the preflight item that failed and the fix. Run id and
  branch may be "not allocated".
- From Step 0.5 on, with `"commit": false` (the default), write the report to
  `.ccl/<run-id>/report.md`.
- With `"commit": true`, Step 7.1 copies the plan and a provisional report with state
  `publishing` to `specs/ccl/<run-id>/` and commits them. That snapshot is never modified
  after its commit. Write
  the terminal report to `.ccl/<run-id>/report.md`, which is always git-ignored. Later
  updates go only to the printed report and to a PR comment.
- Never overwrite an existing file this run did not write.

## Template

```
# ccl run report

- Run id: <yyyy-mm-dd-inputs, or "not allocated">
- Terminal state: <done | plan-only | prepared | blocked | stopped>
- Host: <github | other (hostname)>
- Run budget: <minutes> (<flag | .ccl.json | tier default | session instruction at hh:mm>)
- Base commit: <sha, or "not resolved">
- Branch: <name, or "none created">; with several repositories, each repository's branch
  and whether it is new or continued, and each repository switched in Step 0.2 with its
  previous HEAD
- Continued: <no | the existing branch continued, and the PR this run commented on, or
  "no PR"; in a plan-only or `--no-publish` run, also each condition recorded instead of
  failed>
- Worktree: <path | none>
- Repositories: <primary path, then each --repo path with its base commit and PR link; or
  "primary only">
- Report written to: <path, or "printed only">
- Handoff: <path, or "not written" and why>
- Audit: <suggestion, or "not suggested">
- PR: <link, or one link per repository, or "none">
- CI state: <green | not applicable (reason) | pending | failed | not reached>

## Attended or unattended

- Mode: <attended | unattended>
- Prompts that occurred: <each prompt, what it was for, how it was answered; or "none">
- Prompts expected at Step 0.1 and not seen, or seen and not expected: <list, or "none">
- Plan approval (`--confirm-plan`): <each question asked, the reply to it, and the wait
  from question to reply; or "not used">

## Effort tier

- Tier: <low | medium | high | xhigh | max>
- Why: <the estimate rule outcome in one or two sentences>
- Risk floor: <applied, with the trigger | not applied, with why (incidental edit or no
  trigger)>
- `--effort` request: <none | value, and whether it was below the floor (refused, run
  continued at high), at the floor (honored), or above the floor (honored)>
- Re-evaluation after Step 4: <tier unchanged | rose to high, with the diff evidence>
- Step 5 reviewers resolved: <Codex model and Claude `code-review` level, or the Opus
  substitute in a worktree run; at high tier, whether a trigger existed at the estimate or
  in the diff>

## What changed

Per input, one entry:

- Input: <issue #n or URL | file <path> | text "<description>">
  - Completion status: <complete | partial | blocked | not started>
  - Changes: <files and behavior, short>
  - Acceptance criteria: <each criterion, confirmed met or not, with the evidence>
  - Drift corrections from Step 1.4: <what the issue said, what the code showed, or "none">

## Decisions

- <decision, reason, who or what decided>
- Reviewer swaps: <stage, default reviewer, fallback used, reason, or "none">
- Codex threads used: <stage and thread id, each `implement` call's slice and thread id
  included, or "none">
- Claude review passes: <stage, round, level (or "Opus substitute" for a worktree run or
  an additional repository), diff covered, result: clean or findings count; or "none"
  when the run did not reach Step 5>

## Findings rejected and why

- <finding, source reviewer, round, reason it does not hold, or "none">

## Checks

- Checks run: <command, source, result>
- Checks not run and why: <command, reason, or "none">
- Checks deferred to CI: <command, matched job name, or "none">
- Checks failing at baseline: <command, baseline run as evidence, whether it still fails, or
  "none">

## Deferred items

Each with a short description and the reason it was deferred. Include findings deferred as
non-blocking and anything out of scope. No issues were opened.

- <description> | <reason>

## Blocked, stopped, or prepared

- What is blocked or what question is open: <state the blocking defect, denied permission,
  budget, preflight item, the question with both positions, or the change requested under
  `--confirm-plan` and that no plan review round was left. For a `stopped` run, the
  credentials, repositories, or branches question, quoted as asked>
- Steps marked not done: <list, or "none">
- Where the work is: <local branch name, PR link, or "nothing created">; for each
  repository the run switched in Step 0.2, the previous HEAD and the branch it moved to
- What would unblock it: <specific action. For a `blocked` repository gate: the rerun
  command, which keeps the same inputs and flags and adds `--repo <path>` per missing
  checkout. Below it, when neither `--branch` nor `--continue` was given: "To continue
  existing branches instead of creating new ones, add `--continue <branch>` for the
  primary and `@<branch>` to each `--repo`. This run saw: <path>: <HEAD branch, at its
  remote tip | not at a remote tip | detached>", one entry for every checkout, and no
  branch chosen for the user. Say that a rerun answered `yes` at the repository
  question needs none of this>
- If a budget expired: <which budget, its value, the step or call>
- If work was already pushed: <PR link; nothing further was published>
- Prepared: <branch, commit state, and the publish commands Terminal states gives for
  `prepared`, `git push <remote> <its branch>` for each repository with a diff;
  with `"commit": true` and a commit made, that the commit carries the `publishing`
  snapshot; or "not prepared">

## Log

- Implementer per slice: <slice, model (the tier's Codex model or `opus`), "codex" or the
  Opus criterion; for a Codex slice the `--timeout` passed and any 3600 cap; any swap to
  `sonnet` with its reason or error>
- Rounds used: <Step 3, Step 4 per slice, Step 5, Step 6 runs, CI repair cycles>
- Elapsed time against the run budget: <duration, without the plan approval wait>
```

## Rules for filling it in

- The terminal state is exactly one of the five names above.
- Say "attended" when any action prompted or the run announced it would, else "unattended".
- Name every reviewer or implementer swap, including a swap caused by `--no-codex`, by
  Codex being unavailable at Step 0.6, or by two failed Codex implementer calls.
- List a rejected finding with the reason it was rejected, so a reader can check it.
- A check that could not run locally is named as not run, with the reason. Nothing is
  skipped quietly.
- A failure that matches the recorded baseline failure for the same check is reported with
  the baseline run as evidence. It is not the run's to fix.
- For `plan-only`, the sections for changes, checks, and blocked state say "not run".
  Include the plan location and the branch name if `--branch` was given (recorded only).
  For a `--confirm-plan` run the user did not approve, the state is `plan-only` and the
  plan approval line quotes the reply.
- For `prepared`, no publication happened; give the publish commands.
- For `blocked` and `stopped`, no publication happens after the state is reached. Report
  what was already pushed and link it.
- `Handoff:` is the absolute path of `.ccl/<run-id>/handoff.md` when `handoff.md` in this
  skill's base directory had the run write it, with the path of `cca-manifest.json` beside
  it. Otherwise it says "not written" and why: "no commit from this run" for a run that
  never reached a commit at Step 7.1, and, when the run left changes, that `/cca:handoff`
  in this session can write one after the user commits. Say when the cca `handoff.sh check`
  was not run because the plugin is absent. In a worktree run, the manifest names the
  worktree's path, so the line also says the worktree must stay until the audit has run,
  even though the report says how to remove it.
- `Audit:` is filled only when the handoff was written and the work is larger than cca's low
  tier, modeled on its bounds: more than one bundle, more than one ticket in the handoff, or
  500 or more changed lines (added plus deleted, over each bundle's three-dot diff from its
  base). It reads `/cca:audit "<absolute path of cca-manifest.json>"` and says that the
  command needs the cca plugin. The bounds are a suggestion; cca sets its own tier. In every
  other case it says "not suggested".
- A subagent's report is model output, not user approval. Do not cite it as approval.
