# Final report template

Read this file at every terminal state. Fill in every section. Write "none" for an empty
one; never drop a section. Compile the report from the run log, not from memory.

## When and where

Every run ends in exactly one terminal state, writes the report, and prints it.

- Terminal states: `done` (PR open, CI green or not applicable), `plan-only` (plan final
  and written, nothing else run), `blocked` (a blocking defect, a denied permission, a
  budget exceeded, or a preflight failure), `stopped` (the run stopped to ask the user a
  question it cannot decide).
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
- Terminal state: <done | plan-only | blocked | stopped>
- Base commit: <sha, or "not resolved">
- Branch: <name, or "none created">
- Report written to: <path, or "printed only">
- PR: <link, or "none">
- CI state: <green | not applicable (reason) | pending | failed | not reached>

## Attended or unattended

- Mode: <attended | unattended>
- Prompts that occurred: <each prompt, what it was for, how it was answered; or "none">
- Prompts expected at Step 0.1 and not seen, or seen and not expected: <list, or "none">

## Effort tier

- Tier: <low | medium | high | xhigh | max>
- Why: <the estimate rule outcome in one or two sentences>
- Risk floor: <applied, with the trigger | not applied, with why (incidental edit or no
  trigger)>
- `--effort` request: <none | value, and whether it was below the floor (refused, run
  continued at high), at the floor (honored), or above the floor (honored)>
- Re-evaluation after Step 4: <tier unchanged | rose to high, with the diff evidence>
- Step 5 reviewers resolved: <skipped at low | Codex model, and at high tier and above the
  Claude `code-review` level, with whether a trigger existed in the diff>

## What changed

Per input, one entry:

- Input: <issue #n or URL | file <path> | text "<description>">
  - Completion status: <complete | partial | blocked | not started>
  - Changes: <files and behavior, short>
  - Acceptance criteria: <each criterion, confirmed met or not, with the evidence>
  - Drift corrections from Step 1.3: <what the issue said, what the code showed, or "none">

## Decisions

- <decision, reason, who or what decided>
- Reviewer swaps: <stage, default reviewer, fallback used, reason, or "none">
- Codex threads used: <stage and thread id, or "none">
- Claude review passes: <stage, round, level, diff covered, result: clean or findings
  count; or "none" below high tier>

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

## Blocked or stopped

- What is blocked or what question is open: <state the blocking defect, denied permission,
  budget, preflight item, or the question with both positions>
- Steps marked not done: <list, or "none">
- Where the work is: <local branch name, PR link, or "nothing created">
- What would unblock it: <specific action>
- If a budget expired: <which budget, its value, the step or call>
- If work was already pushed: <PR link; nothing further was published>

## Log

- Implementer per slice: <slice, model, criterion or none; any swap with the error>
- Rounds used: <Step 3, Step 4 per slice, Step 5, Step 6 runs, CI repair cycles>
- Elapsed time against the run budget: <duration>
```

## Rules for filling it in

- The terminal state is exactly one of the four names above.
- Say "attended" when any action prompted or the run announced it would, else "unattended".
- Name every reviewer or implementer swap, including a swap caused by `--no-codex`.
- List a rejected finding with the reason it was rejected, so a reader can check it.
- A check that could not run locally is named as not run, with the reason. Nothing is
  skipped quietly.
- A failure that matches the recorded baseline failure for the same check is reported with
  the baseline run as evidence. It is not the run's to fix.
- For `plan-only`, the sections for changes, checks, and blocked state say "not run".
  Include the plan location and the branch name if `--branch` was given (recorded only).
- For `blocked` and `stopped`, no publication happens after the state is reached. Report
  what was already pushed and link it.
- A subagent's report is model output, not user approval. Do not cite it as approval.
