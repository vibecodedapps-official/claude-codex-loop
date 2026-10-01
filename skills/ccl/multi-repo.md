# Multi-repo mode

Read this file at the start of Step 0, before host detection, when `repos` is not
`none`. It is also read on adoption at Step 1.2, when a reply adds repositories; see
Adoption at Step 1.2 below. It holds the rules for a run that writes to additional
checkouts, and they apply for the rest of the run.

`repos` names additional writable checkouts. The current checkout is the primary. Read-only
repositories are not named; agents read them directly. Every rule below changes the step it
names, for each repository, and leaves the rest of that step as written.

- Host: every repository must be on the same host as the primary: the same class and the
  same hostname. Classify each with Step 0 host detection, run as `git -C <path>`. A
  `--repo` checkout on a different host, including a GitHub Enterprise host beside a
  github.com primary, is a preflight failure naming both hostnames. When the hostname is
  not `github.com`, every `-R` argument takes the form `<host>/<owner>/<repo>`. On `other`,
  all repositories end in `prepared` together.
- Issue references: a bare `#n` always names an issue of the primary. An issue of a `--repo`
  checkout must be given as a full URL.
- Worktree: the Step 0.3 worktree exception is not available. A primary with a clean status
  and skip-worktree or assume-unchanged files that differ from HEAD ends in `blocked`, and
  the report says so. This keeps Step 5's `codex-lite:review` and `code-review` of the
  primary correct.
  Step 0.3's flagged-file check runs per repository. An additional repository whose
  skip-worktree or assume-unchanged files differ from `HEAD` does not block: the run
  continues and records those paths in `run.md` under that repository. The report names
  them as local state that repository's Step 3.7.3 baseline and Step 6 checks ran against.
  Its review is unaffected, because its patch comes from `git -C <path> diff <base>`,
  which leaves those paths out. No slice may edit such a path: an edit to it is invisible
  to `git diff`, `git add -N`, and staging, so review and the commit would drop it while
  local checks pass on it. If the plan needs to change one, the run ends in `blocked` at
  Step 2 naming the path. Implementer prompts for that repository name the paths as off
  limits.
- Commit snapshot: with `"commit": true`, the `specs/ccl/<run-id>/` snapshot is committed in
  the first repository, the primary first and then the `--repo` order, that has a diff. A
  repository with no diff never receives it and gets no PR.
- `.ccl.json`: the primary's governs `commit` and `timeouts`. Each repository's own `checks`
  list is read for that repository's Step 3.7.3 and Step 6.
- `gh` and `git` targets: every `gh` call for an additional repository (PR create and edit,
  checks, CI reads under Step 7.3.1, and issue comments) runs inside one Bash call as `cd
  <path> && ...`, or with `-R <owner>/<repo>` where the subcommand accepts it. `gh api` does
  not accept `-R`: run it from the checkout, or spell the endpoint out as
  `repos/<owner>/<repo>/...` instead of `repos/{owner}/{repo}/...`. Every `git` call for it
  runs as `git -C <path>`.
- PR body files: each repository's body is written to `.ccl/<run-id>/pr-body-<slug>.md` in
  the primary (the primary's own may stay `pr-body.md`). Every `--body-file` for an
  additional repository, in `gh pr create`, `gh pr edit`, and the continued-PR `gh pr
  comment`, is an absolute path, because the call runs inside that repository's checkout,
  where `.ccl/<run-id>/` does not exist.
- Step 0.1: read the instruction files and ask-first rules in every repository and union
  them. The permission statement lists each repository's push and PR.
- Steps 0.2 to 0.4 run per repository. Record a base commit and a planning snapshot for
  each.
- `continue`: each repository has its own branch state, `new` or `continue <branch>`.
  The primary's comes from `--branch` or `--continue`; its branch must exist on its
  remote, which the command checks. An additional repository's comes from its
  `@<branch>`, whose form the command checks and whose existence Step 0.2 checks on that
  repository's selected remote (Host detection), a missing one failing preflight; without
  one, from `--continue` when that repository's remote has that name, checked with
  `git -C <path> ls-remote --heads <remote> refs/heads/<branch>`; else `new`. The branch
  question (Adoption at Step 1.2) can set the state of a repository that has no explicit
  one.
  A repository that continues a branch follows Step 0.2 for it, including the `HEAD`
  requirement and the consented switch. A `new` repository creates its branch in Step
  3.7.2 under the naming rule, with the collision check. Branch names may differ
  between repositories. Each repository's state is recorded in `run.md`.
- Ignoring `.ccl/`: write the exclude in every repository. Artifacts live only in the
  primary's `.ccl/<run-id>/`, with a section per repository in `inputs.md` and `run.md`.
- Input guard: an issue may belong to any listed repository. Compare the issue URL's owner
  and repo with `git -C <path> remote -v` of each listed repository, and record which one.
- Step 0.5: fetch each issue from its owning repository with `gh issue view <n> -R
  <owner>/<repo>`.
- Step 2: name the repository of every slice. No slice spans repositories.
- Step 3.7: 3.7.2 switches or creates per repository with its own branch state. A
  repository that continues a branch switches to it and skips the name check. A `new`
  repository creates its branch under the naming rule, with the collision check run in
  that repository. 3.7.3 runs a baseline per repository.
- Step 4: each implementer prompt names the repository path of its slice as the only
  checkout it edits. A Codex implementer call for an additional repository passes `--cwd
  <absolute path of that repository>` as its last option, and its request names files by
  absolute path or carries the content inline.
- Step 5.2: review only repositories that have a diff from their base commit (`git -C <path>
  diff <base> --stat`, after `git -C <path> add -N` of new files). A repository with an
  empty diff is skipped and named in `run.md`.
  - Codex: write `.ccl/<run-id>/diff-<slug>.patch` from `git -C <path> diff <base>` for
    every additional repository with a diff, in both cases below, because the fallback
    subagent and the Claude substitute read it. When the primary has a diff, review it
    with `codex-lite:review --base <primary base>`.
    - After a `drop` answer, the Reviewer contract item 5 diff scan runs before each
      native `codex-lite:review` call, the primary's and each `--cwd` one, over `git -C
      <path> diff <base>` of that repository, after the `add -N` of new files. On a match
      that repository is reviewed through `codex-lite:ask` with its patch file
      (`diff.patch` or `diff-<slug>.patch`), the matched values replaced by `<redacted:
      key>`, in a fresh thread that becomes that repository's thread (for the primary,
      the stage's thread), recorded in `run.md`; its follow-ups resume that thread. Below
      0.9.0 the additional repositories already go through `codex-lite:ask` with a
      patch, and only the replacement applies to them. Every patch file written after
      `drop` carries the same replacements.
    - With codex-lite 0.9.0 or later, as recorded in Step 0.6: review each additional
      repository with a diff by `codex-lite:review --base <its base> --model <id>
      --timeout <s>` on the first line and `--cwd <absolute path of that repository>` on
      the second, resolved as the Step 4 implementer rule does, a fresh thread
      per repository. `run.md` records a thread id per repository. A Step 5.4 follow-up
      for that repository resumes that repository's thread with `codex-lite:ask --resume
      <its thread>`, naming `diff-<slug>.patch`, never another repository's thread.
    - Below 0.9.0, codex-lite reviews only the session's checkout. The primary's review
      thread becomes the stage's thread. When no primary review happened, the first
      additional repository's patch review starts the stage's thread, as a fresh
      `codex-lite:ask` thread. Every later additional repository resumes it with
      `codex-lite:ask --resume <stage thread>`, naming its patch file.
  - Under `--no-codex` or after a swap, the stage's fallback subagent (Codex availability
    item 3) is given every repository's patch file, `diff.patch` for the primary and
    `diff-<slug>.patch` for each additional repository, instead of reading `git diff
    <base-commit>` itself. Before the first fallback review, write the primary's diff to
    `.ccl/<run-id>/diff.patch`, after `git add -N` of new files. Step 5.4 and CI repair
    (Step 7.3.5) continue that same subagent with SendMessage.
  - Claude, at every tier: the `code-review` pass covers the primary when it has a diff,
    as the Claude review contract says. For each additional repository with a diff, the
    Claude slot is a Claude subagent at Agent model `opus`, given the repository's patch
    file, the acceptance criteria, and the reply shape of Reviewer contract item 8, and
    told to read and report only. This is a defined substitute for a checkout the skill
    cannot target, and a worktree run uses the same substitute for its checkout (Claude
    review contract item 7). It is not a swap. Record it in `run.md` per repository and
    name it in the report. It is the one Claude pass that is continued rather than fresh:
    Step 5.4 follow-ups continue the same subagent with SendMessage.
- Step 6: discover and run checks per repository.
- Step 7: commit and push per repository that has a diff, each to its own branch, and open
  one PR per such repository. Each body has a "Related pull requests" section, with
  "pending" there in the first PR opened. Then edit each body once with `gh pr edit <n>
  --body-file` to fill the sibling links. With `continue`, a continued PR gets no body
  edit: its comment is posted after every PR of the run is open, so its "Related pull
  requests" section is filled from the start, and the `gh pr edit` pass covers only the
  PRs this run opened. Watch CI per PR, and comment on each issue with every PR link.
  `done` needs every PR green. A `blocked` in any repository blocks the run, and no
  further publication happens in any repository, with one exception: the sibling-link
  edit of a PR this run already opened still runs, so no PR is left saying "pending".
  The exception covers only PRs this run opened; a continued PR whose comment was not
  yet posted gets none. Editing the body of this run's own PR is inside the approval
  scope and publishes nothing new.
- Step 7.3.5: a CI repair review for an additional repository uses the Step 5.2 rule for
  the recorded codex-lite version. With 0.9.0 or later, Codex resumes that repository's
  own thread through `codex-lite:ask --resume`, naming `diff-<slug>.patch`, never another
  repository's thread. Below 0.9.0, it uses the stage thread with the patch, never
  `codex-lite:review`. The repository's Claude subagent is continued in both cases.
- Closing references: an issue is closed only by the PR in its own repository (`Closes #n`).
  Every other PR of the run cites it as `Refs <owner>/<repo>#n`. The issue status comment
  names the PR in the issue's repository first, then the siblings.

## Adoption at Step 1.2

Step 1.2 reads this section when the reply to its repository question adds repositories
to `repos`. From the reply on, the run is Multi-repo mode with the rules above. After the
reply, do these in order, across all adopted repositories, not one repository at a time:

1. Host detection for each adopted repository. A host mismatch is a preflight failure.
2. Step 0.1 for each: the instruction files and ask-first rules, unioned with those
   already read. Print the permission statement again, with each repository's push and
   PR.
3. Step 0.2 for each: the default branch and the base commit.
4. Step 0.3 for each: the clean-tree check. A dirty adopted repository ends in `blocked`
   naming it. Never stash.
5. The branch question below.
6. Step 0.2's `continue` checks for each repository that continues a branch, the primary
   included when the reply set its branch, in this order: fetch the branch with the
   explicit refspec; replace the base commit with `<remote>/<branch>`; run Step 0.2's
   `HEAD` check, with the consented switch when `HEAD` is not at the new base or the
   local branch is behind it; revise
   the snapshot note Step 0.5 wrote to `run.md`. Step 1.3 verifies against the new base.
7. Step 0.4 snapshot for each, and the `.ccl/` exclude for each.
8. The sections for each repository in `inputs.md` and `run.md`. Record "Multi-repo mode
   adopted at Step 1.2 by reply" in `run.md`.

### The branch question

Ask it once, after item 4, for every repository with no explicit branch state: each
adopted repository, and the primary when neither `--branch` nor `--continue` was given.
Explicit values are never changed by this question. The run does not infer intent from
checkout state; it suggests, and the reply decides.

Per repository, the suggestions are:

- The branch HEAD is on, when its tip equals `<remote>/<that branch>` and it is not the
  default branch.
- Every remote branch B, from `git -C <path> ls-remote --heads <remote>`, whose common
  stem with a seed name A of another repository qualifies. The seed names are each
  repository's explicit branch and its suggestion from the rule above, computed first
  for every repository. This rule runs once over them, and a branch it suggests is never
  itself a seed. The common stem is the longest common prefix of A and B when that
  prefix is the whole of A or of B, and otherwise that prefix cut back to its last
  separator character (`/`, `-`, `_`, or `.`), the separator excluded. It qualifies when
  it is not empty, was not cut at a `/`, and is not a prefix of this repository's
  default branch name. So a shared directory such as `feature/user/` alone never
  qualifies, and this rule never suggests the default branch.

For example, the primary is on `feature/user/2026.09-migration-fixes` and a third
repository on `feature/user/2026.09-migration-etl`, each at its remote tip. The second
repository is on an unrelated branch at its remote tip and has a remote branch
`feature/user/2026.09-migration-heart`. Its common stem with either of the other names is
`feature/user/2026.09-migration`, which qualifies, so the second repository's line
suggests its `HEAD` branch and `feature/user/2026.09-migration-heart`. A repository with
two suggestions is not covered by `yes`, so the reply names the choice in a
`<path>@<branch>` line.

Print one line per repository, `<path>: suggested <branch>[, <branch>]` or `<path>: no
suggestion`, then ask:

> Which branch does each repository continue? Reply `yes` on the first line to take the
> suggestion in every repository that has exactly one, then one `<path>@<branch>` or
> `<path>@new` line for each other repository. Lines alone, without `yes`, are also
> accepted. A repository with no suggestion or several must have a line. A checkout that
> is clean and not at its branch tip is switched to it.

The reply must cover every repository the question lists. Validate each chosen branch
as the command does for `--continue`: `git check-ref-format --branch`, then `git -C
<path> ls-remote --heads <remote> refs/heads/<branch>` on that repository's selected
remote; a missing branch is a failure, never a fallback to creating it. `@new` sets
the state `new`. A clear reply continues. Anything else, including a reply that does not
cover every repository or names a missing branch, ends in `stopped` with the question in
the report, under the Questions mechanic in Mechanics, and the wait is outside the run
budget. After the reply, rerun Step 0.3's clean-tree check on every repository the run
edits.

For an adopted repository, and for the primary when the reply set its branch, the reply
is the consent for the switch in Step 0.2; no further question is asked.
