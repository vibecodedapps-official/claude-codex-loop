# Handoff and audit manifest

Read this file in Final report handling, before the report is filled, when the run has a
commit of its own. It says when the run writes a handoff and an audit manifest, from what,
and in what shape. Both are written from the run's record, never from memory.

The handoff format's home is the cca plugin's `skills/cca/handoff.md`. This file restates
only what ccl needs. The result must pass the cca plugin's `handoff.sh check`. To find the
script, run `claude plugin list --json` and take the entry whose `id` starts with `cca@`:
the script is `<installPath>/skills/cca/scripts/handoff.sh`, where `installPath` is that
entry's. cca is supported only when that script exists, which is cca 0.2.0 or later. Run
`sh <script> check <path of handoff.md>`, which prints `handoff: ok` on success, and fix
what it names. With no such entry the plugin is absent: follow the shapes below exactly
and say in the report that the check was not run. With the entry but no script, treat it
the same way and say in the report that the check was not run because the installed cca
is older than 0.2.0.

The same entry gates the ticket keys `parent` and `links`, which cca 0.3.0 added. Check it
once per run: the gate passes only when the entry's `version` parses as numeric
`major.minor.patch` and is 0.3.0 or later. Compare the three parts as numbers, never as
strings: 0.10.0 is later than 0.3.0. With no entry, an older version, or one that does not
parse, such as a commit hash, no ticket gets either key and the reads of "Parent and
links" are skipped. When the run has an issue input, `run.md` records that they were
skipped with the installed version or that no `cca@` entry was found.

## When

Write both files whenever the run has at least one commit of its own (Step 7.1) in some
repository, whatever the terminal state: `done`, a `prepared` run whose push was withheld
after Step 7.1 committed, and `blocked` or `stopped` after Step 7.1.

A run with no commit of its own writes neither: `plan-only`, `--no-publish` before Step
7.1 committed, and any state before Step 7.1. The report's `Handoff:` line says "not
written: no commit from this run". When the run left changes and cca is supported (the
`cca@` entry and its script both exist, as above), it adds that `/cca:handoff` in this
session can write one after the user commits; otherwise it says that `/cca:handoff`
needs cca 0.2.0 or later. Never commit to make a handoff possible.

Both files go in `.ccl/<run-id>/`: `handoff.md` and `cca-manifest.json`. They are never
committed. Never overwrite a file this run did not write.

## Sources

Only the run's durable files: `inputs.md`, `plan.md` with its review log, `run.md`, the
`pr-body.md` the run wrote, and git. The parent and closing PRs come from the reads that
"Parent and links" records in `run.md`. A value those files do not hold is written as the
format's `none`, `none recorded`, or `not recorded`. Never invent an option, a reason, an
owner, or a check.

The decision record in `run.md` is one entry per decision: the choice, its reason, the
options weighed with why each was rejected (from the plan review log), who decided, and
where it was published (the PR body or a PR comment, with its URL, once Step 7.2 publishes
it). Who decided is `user` when the user answered it, `plan approval` when the user
approved a plan containing it under `--confirm-plan`, `review` when the run took a
reviewer's recommendation, else `run`.

## Parent and links

In Final report handling, before the handoff is written, read each issue input's parent
and closing PRs, only when the gate above passes and only on the `github` host. Read each
distinct issue once: `#n` and its URL are one ticket. A file or text input causes no read.
The reads come this late so that the run's own PR is included when GitHub lists it as
closing the issue; `links` records what the forge lists, nothing more. Read by the issue's
URL, so an issue of an additional repository works:

- Closing PRs: `gh issue view <URL> --json closedByPullRequestsReferences`. Each entry's id
  is `github:<repository.owner.login>/<repository.name>#<number>`. The field needs gh
  2.73.0 or later; an older gh fails the read.
- Parent:
  `gh api graphql -f query='query{repository(owner:"<owner>",name:"<repo>"){issue(number:<n>){parent{number repository{nameWithOwner}}}}}'`,
  with `--hostname <host>` when the issue URL's host is not `github.com`. `"parent":null`
  means no parent. A parent's id is `github:<repository.nameWithOwner>#<number>`.

Record the result in `run.md` per issue: the parent or none, and the closing PRs or
none, with a failed read named in place of its result, with its error line. The handoff
is written from that record only.

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
are `- <key>: <value>` at column 0, each once (an optional key at most once), in the
order below. A list key (`commits`, `verified`, `options`, `links`) has an empty value
followed by entries indented exactly two spaces (`  - `), or the value `none`
(`none recorded` for `options`). No credentials anywhere in the file. A model, agent, or
tool is never `decided_by`; elsewhere, paths, commit subjects, and check commands are
copied as recorded, tool names included.

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
- `base`: always `<selected remote>/<base branch>`, where the remote is that repository's
  selected remote and the base branch is the PR's base branch when a PR exists, else the
  default branch. cca refreshes only a remote-tracking base.

### Tickets

One `### <ticket id>` per distinct input.

- The id of an issue input is `github:<owner>/<repo>#<n>`. An alias of the same issue, `#n`
  and its URL, is one ticket. The id of a file or text input is `<run-id>/input-<k>`, k
  from 1 in input order. Ids are unique across the file.
- Keys, in order: `type`, `state`, `iteration`, `owner`, `parent`, `links`, `bundles`,
  `problem`, `decision`, `commits`, `verified`. `parent` and `links` are optional.
- `type`: `issue`, or `task` for a file or text input. `state`: the issue's state, else
  `none`. `iteration`: the issue's milestone title, else `none`. `owner`: the login of the
  issue's first assignee, else `none`.
- `parent` and `links`, from the reads recorded in `run.md`, only for an issue input; a
  file or text input gets neither. `parent`: `github:<owner>/<repo>#<n>` when the read
  found a parent; no key when the issue has none or the read failed. `links`: an empty
  value with `  - closed by: github:<owner>/<repo>#<n>` per distinct closing PR, in the
  order gh returned them, duplicates dropped; `none` only when the read succeeded and
  returned no PR; no key when the read failed. Never write `links: none` for links that
  were not read. With the gate failed, no ticket has either key.
- `bundles`: every bundle with a commit of this run attributed to the input, comma
  separated; an input with none lists the first bundle.
- `problem`: Step 1's reading of the input. `decision`: the plan's approach for it.
- `commits`: `  - <bundle> <sha>: <commit subject>` for each commit this run made, from each
  repository's base commit in `run.md` to its head, under the input its message names; in
  a one-input run, every commit; `none` when the input has none. The sha is 7 to 40
  lowercase hex digits. A commit on a continued branch from before this run is not listed
  as a claim; the audit still covers it, and cca groups it by its own rules.
- `verified`: `  - <check> passed in <bundle> (<directory>); check: <command>` for each
  Step 6 check that ran and passed in the repository of the ticket's first bundle, where
  `<bundle>` is that bundle and `<directory>` is the absolute path the check ran in;
  `none` when no check passed there. Step 6 runs per repository in Multi-repo mode. A
  check from another bundle of the ticket is not listed, because cca attributes every
  verified entry to the first bundle.

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
  `owner`: `none`. A raised ticket has no forge record, so it gets neither `parent` nor
  `links`.
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
  the handoff, with the same `base` value, `<selected remote>/<base branch>`. cca audits a
  PR at its remote head, so a PR whose head lacks this run's commits (a declined push,
  or an unpushed CI repair) is named by its local branch instead.
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

A made-up run with one issue, one repository, and one deferred item. It assumes cca 0.3.0
or later; with an older cca the `parent` and `links` lines are absent.

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
- parent: github:acme/widget-app#9
- links:
  - closed by: github:acme/widget-app#31
- bundles: widget-app
- problem: The sync job retries a failing call forever and never reports an error.
- decision: Cap the retries at five with a growing delay, then raise the last error.
- commits:
  - widget-app 4e1a9c2: cap the sync retries and raise the last error
- verified:
  - The unit tests passed in widget-app (/home/dev/widget-app); check: npm test

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
