# Handoff and audit manifest

Read this file in Final report handling, before the report is filled, when the run has a
commit of its own. It says when the run writes a handoff and an audit manifest, from what,
and in what shape. Both are written from the run's record, never from memory.

The handoff format's home is the cca plugin's `skills/cca/handoff.md`. This file restates
only what ccl needs. The result must pass the cca plugin's `handoff.sh check`. To find the
script, run `claude plugin list --json` and take the entry whose `id` starts with `cca@`:
the script is `<installPath>/skills/cca/scripts/handoff.sh`, where `installPath` is that
entry's. Run `sh <script> check <path of handoff.md>`, which prints `handoff: ok` on
success, and fix what it names. With no such entry the plugin is absent: follow the shapes
below exactly and say in the report that the check was not run.

## When

Write both files whenever the run has at least one commit of its own (Step 7.1) in some
repository, whatever the terminal state: `done`, a `prepared` run whose push was withheld
after Step 7.1 committed, and `blocked` or `stopped` after Step 7.1.

A run with no commit of its own writes neither: `plan-only`, `--no-publish` before Step
7.1 committed, and any state before Step 7.1. The report's `Handoff:` line says "not
written: no commit from this run". When the run left changes, it adds that `/cca:handoff`
in this session can write one after the user commits. Never commit to make a handoff
possible.

Both files go in `.ccl/<run-id>/`: `handoff.md` and `cca-manifest.json`. They are never
committed. Never overwrite a file this run did not write.

## Sources

Only the run's durable files: `inputs.md`, `plan.md` with its review log, `run.md`, the
`pr-body.md` the run wrote, and git. A value those files do not hold is written as the
format's `none`, `none recorded`, or `not recorded`. Never invent an option, a reason, an
owner, or a check.

The decision record in `run.md` is one entry per decision: the choice, its reason, the
options weighed with why each was rejected (from the plan review log), who decided, and
where it was published (the PR body or a PR comment, with its URL, once Step 7.2 publishes
it). Who decided is `user` when the user answered it, `plan approval` when the user
approved a plan containing it under `--confirm-plan`, `review` when the run took a
reviewer's recommendation, else `run`.

## Mapping

The file is the cca format: the frontmatter, then the four sections in this order, each
once.

```
---
cca-handoff: 1
generated: <ISO 8601 time>
---

# Handoff: <title>

## Bundles
## Tickets
## Decisions
## Raised tickets
```

`generated` is the time the file is written. The title is one line naming the run's work.
A section with no items holds the single line `none`; `## Bundles` always has an item,
since a commit of this run exists. Every value is on one line and holds no tab. Item keys
are `- <key>: <value>` at column 0, each once, in the order below. A list key (`commits`,
`verified`, `options`) has an empty value followed by entries indented exactly two spaces
(`  - `), or the value `none` (`none recorded` for `options`). No credentials anywhere
in the file. A model, agent, or tool is never `decided_by`; elsewhere, paths, commit
subjects, and check commands are copied as recorded, tool names included.

### Bundles

One line per repository with at least one commit from this run, in Multi-repo order
(primary first):

```
- <name>: repo <absolute path>; pr <link or none>; branch <branch>; base <base ref>
```

- Name: the repository directory's base name, lowercased, each run of characters outside
  `a-z0-9._-` turned into `-`, then `r-` in front when it does not start with `a-z0-9`,
  then `-2`, `-3` on a clash with any name already used, in order.
- `repo`: the absolute path of the checkout; in a worktree run, the worktree's path. The
  manifest names that path too, so the worktree must stay until the audit has run.
- `pr`: the PR as `github:<owner>/<repo>#<n>`, or `none`.
- `branch`: that repository's branch.
- `base`: that repository's PR base branch, else `<selected remote>/<default branch>`, the
  ref ccl fetched.

### Tickets

One `### <ticket id>` per distinct input.

- The id of an issue input is `github:<owner>/<repo>#<n>`. An alias of the same issue, `#n`
  and its URL, is one ticket. The id of a file or text input is `<run-id>/input-<k>`, k
  from 1 in input order. Ids are unique across the file.
- Keys, in order: `type`, `state`, `iteration`, `owner`, `bundles`, `problem`, `decision`,
  `commits`, `verified`.
- `type`: `issue`, or `task` for a file or text input. `state`: the issue's state, else
  `none`. `iteration`: the issue's milestone title, else `none`. `owner`: the login of the
  issue's first assignee, else `none`.
- `bundles`: every bundle with a commit of this run attributed to the input, comma
  separated; an input with none lists the first bundle.
- `problem`: Step 1's reading of the input. `decision`: the plan's approach for it.
- `commits`: `  - <bundle> <sha>: <commit subject>` for each commit this run made, from each
  repository's base commit in `run.md` to its head, under the input its message names; in
  a one-input run, every commit; `none` when the input has none. The sha is 7 to 40
  lowercase hex digits. A commit on a continued branch from before this run is not listed
  as a claim; the audit still covers it, and cca groups it by its own rules.
- `verified`: `  - <check> passed; check: <command>` for each Step 6 check that ran and
  passed; `none` when none ran.

### Decisions

One `### D<n>` per decision record, n from 1. Keys, in order: `ticket`, `decision`,
`rationale`, `options`, `decided_by`, `recorded_at`, `status`.

- `ticket`: the input it concerns. A decision over several inputs or the whole run takes
  the first input in input order. A decision about a deferred item takes that item's
  raised ticket id.
- `options`: `  - chosen: <option>` and `  - rejected: <option>; why: <reason>` entries, or
  `none recorded`. With `status` `taken` or `default taken` and options listed, exactly
  one entry is `chosen:`.
- `decided_by`, by the record's who decided: `role: session user` for `user`; `role:
  session user (plan approval)` for `plan approval`; `checkpoint (recommended option
  taken)` for `review`; `not recorded` for `run`. Never `person:`, since ccl does not
  record the user's name, and never a model, agent, or tool.
- `recorded_at`: `url <link>` when the decision was published, else `checkpoint: <the
  step>`. The audit cannot open a `checkpoint:` pointer.
- `status`: `taken`; `default taken` when no alternative is recorded; `deferred` for a
  decision the run deferred.

### Raised tickets

One `### R<n>` per Deferred item, n from 1. Keys, in order: `ticket`, `type`, `state`,
`iteration`, `owner`, `bundles`, `summary`, `rank`, `in_bundle_confidence`, `reason`.

- `ticket`: `<run-id>/deferred-<k>`. `type`: `task`. `state`: `new`. `iteration` and
  `owner`: `none`.
- `bundles`: the bundle it concerns, else `none`.
- `summary`: its description, on one line. `rank`: its place in report order, from 1.
  `in_bundle_confidence`: `defer`. `reason`: its reason.

## Manifest

`.ccl/<run-id>/cca-manifest.json` is written beside the handoff. It holds one bundle per
line of `## Bundles`:

- `repo`: the absolute path, as in the handoff.
- `pr`: `github:<owner>/<repo>#<n>` when a PR exists and its head on the remote is the
  local head of that repository's branch (`git rev-parse <remote>/<branch>`, which the
  run's own push updated, equals `git rev-parse HEAD`); else `branch` and `base`, as in
  the handoff. cca audits a PR at its
  remote head, so a PR whose head lacks this run's commits (a declined push, or an
  unpushed CI repair) is named by its local branch instead.
- `tickets`: the issue inputs that list that bundle, as `github:` ids.

and a `claims` list holding one entry, the absolute path of `handoff.md`. cca reads it as
an ordinary manifest. A bundle with a PR carries no `branch` or `base`, since cca resolves
them from the PR.

```
{
  "bundles": [
    {"repo": "<path>", "pr": "github:<owner>/<repo>#<n>", "tickets": ["<ticket id>"]},
    {"repo": "<path>", "branch": "<branch>", "base": "<base ref>", "tickets": []}
  ],
  "claims": ["<absolute path of handoff.md>"]
}
```

## Example

A made-up run with one issue, one repository, and one deferred item.

```
---
cca-handoff: 1
generated: 2026-10-01T09:30:00Z
---

# Handoff: add a retry limit to the sync job

## Bundles

- widget-app: repo /home/dev/widget-app; pr github:acme/widget-app#31; branch ccl/sync-retry; base origin/main

## Tickets

### github:acme/widget-app#12
- type: issue
- state: OPEN
- iteration: Sprint 4
- owner: dev-one
- bundles: widget-app
- problem: The sync job retries a failing call forever and never reports an error.
- decision: Cap the retries at five with a growing delay, then raise the last error.
- commits:
  - widget-app 4e1a9c2: cap the sync retries and raise the last error
- verified:
  - The unit tests passed; check: npm test

## Decisions

### D1
- ticket: github:acme/widget-app#12
- decision: Cap the retries at five.
- rationale: Five attempts cover a short outage without holding a worker for minutes.
- options:
  - chosen: Five retries with a growing delay
  - rejected: Retry until the call succeeds; why: A permanent failure would never surface
- decided_by: role: session user (plan approval)
- recorded_at: url https://github.com/acme/widget-app/pull/31
- status: taken

## Raised tickets

### R1
- ticket: 2026-10-01-12/deferred-1
- type: task
- state: new
- iteration: none
- owner: none
- bundles: widget-app
- summary: Make the retry limit configurable.
- rank: 1
- in_bundle_confidence: defer
- reason: The issue asks only for a cap, and a setting needs its own design.
```

The manifest for that run:

```
{
  "bundles": [
    {"repo": "/home/dev/widget-app", "pr": "github:acme/widget-app#31", "tickets": ["github:acme/widget-app#12"]}
  ],
  "claims": ["/home/dev/widget-app/.ccl/2026-10-01-12/handoff.md"]
}
```
