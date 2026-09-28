# PR body and issue comment templates

Read this file at Step 7.2, before opening the PR. Fill in every section from the plan, the
run log, and the checks. Write "none" for an empty section.

## PR body

```
## What changed

Per input:

- <input: issue #n, file <path>, or ad-hoc description>: <what changed and where, in a few
  lines. State status: complete or partial. If partial, say what is left.>

## Drift corrections

<For each input whose text had drifted from the code, one line: what the issue said, what
the code showed, and the plan that followed. Write "none" when nothing drifted.>

## Decisions for the reviewer

<Decisions a reviewer needs to understand the shipped change: design choices, options
rejected and why, behavior a reader could take for a bug. Not a log of the run.>

## Checks not run

<Each check that could not run locally, with the reason. Each check deferred to CI, by job
name. Each check failing at baseline, with the baseline run as evidence. "none" if all ran
and passed.>

## Closes

<One line per input, see the rules below.>
```

## Closing reference rules

Decide completion per input after implementation, the reviews the tier requires, and Step 6.
The criteria come from the plan.

- `Closes #n` only when every acceptance criterion for that issue is confirmed met.
- `Refs #n` in every other case: a criterion not met, not confirmed, or an input marked
  partial or blocked. Post a status comment on the issue (see below).
- A file input or an ad-hoc description has no issue to close. List it under "What changed"
  and write no closing line for it.
- Use one line per issue, for example `Closes #12` on one line and `Refs #14` on the next,
  so the platform links each one. Do not join them.

## Issue status comment

Post at Step 7.4, once per source issue, after the PR is open. Post it for `Closes` issues
too, so the issue records the evidence. Never post one before the plan is final.

```
Status from the ccl run <run-id>: <complete | partial | blocked>

PR: <link>

Criteria:
- <criterion>: <met | not met | not confirmed>. <evidence: test name, check result, or
  file and behavior>

Not done:
- <what remains and why, or "nothing">

Deferred:
- <item and reason, or "nothing">
```

Rules for the comment:

- State partial completion plainly. Do not say "done" for a `Refs` issue.
- Cite evidence for each criterion. Do not repeat a claim from a subagent without checking it.
- Do not open new issues. List follow-up work under "Deferred" and in the final report.
- Keep the text free of run-internal paths and artifacts that are not committed.
