# Multi-repo mode

Read this file at the start of Step 0, before host detection, when `repos` is not
`none`. It holds the rules for a run that writes to additional checkouts, and they
apply for the rest of the run.

### Multi-repo mode

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
  which leaves those paths out.
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
- `continue`: the branch must exist on the primary's remote, which the command checks.
  Step 0.2 checks each additional repository's remote with `git -C <path> ls-remote
  --heads <remote> refs/heads/<branch>`. A repository whose remote has the branch
  continues it as Step 0.2 says. A repository whose remote lacks it creates it in Step
  3.7.2 as a new branch from its default branch, as without `continue`. Different branch
  names per repository are not supported.
- Ignoring `.ccl/`: write the exclude in every repository. Artifacts live only in the
  primary's `.ccl/<run-id>/`, with a section per repository in `inputs.md` and `run.md`.
- Input guard: an issue may belong to any listed repository. Compare the issue URL's owner
  and repo with `git -C <path> remote -v` of each listed repository, and record which one.
- Step 0.5: fetch each issue from its owning repository with `gh issue view <n> -R
  <owner>/<repo>`.
- Step 2: name the repository of every slice. No slice spans repositories.
- Step 3.7: 3.7.2 creates the same branch name in every repository, and the name check runs
  in every repository, except that with `continue` a repository whose remote has the
  branch switches to it and skips the name check. 3.7.3 runs a baseline per repository.
- Step 4: each implementer prompt names the repository path of its slice as the only
  checkout it edits.
- Step 5.2: review only repositories that have a diff from their base commit (`git -C <path>
  diff <base> --stat`, after `git -C <path> add -N` of new files). A repository with an
  empty diff is skipped and named in `run.md`.
  - Codex: when the primary has a diff, review it with `codex-lite:review --base <primary
    base>`; that thread becomes the stage's thread. When no primary review happened, the
    first additional repository's patch review starts the stage's thread, as a fresh
    `codex-lite:ask` thread. Every later additional repository resumes it
    with `codex-lite:ask --resume <stage thread>`, naming `.ccl/<run-id>/diff-<slug>.patch`,
    written from `git -C <path> diff <base>`, because codex-lite reviews only the session's
    checkout.
  - Under `--no-codex` or after a swap, the stage's fallback subagent (Codex availability
    item 3) is given every repository's patch file, `diff.patch` for the primary and
    `diff-<slug>.patch` for each additional repository, instead of reading `git diff
    <base-commit>` itself. Before the first fallback review, write the primary's diff to
    `.ccl/<run-id>/diff.patch`, after `git add -N` of new files. Step 5.4 and CI repair
    (Step 7.3.5) continue that same subagent with SendMessage.
  - Claude, at high tier and above: the `code-review` pass covers the primary when it has a
    diff, as the Claude review contract says. For each additional repository with a diff,
    the Claude slot is a Claude subagent at Agent model `opus`, given the repository's patch
    file, the acceptance criteria, and the reply shape of Reviewer contract item 8, and
    told to read and report only. This is a defined substitute for a checkout the skill
    cannot target. It is not a swap. Record it in `run.md` per repository and name it in the
    report. It is the one Claude pass that is continued rather than fresh: Step 5.4
    follow-ups continue the same subagent with SendMessage.
- Step 6: discover and run checks per repository.
- Step 7: commit and push per repository that has a diff, and open one PR per such
  repository. Each body has a "Related pull requests" section, with "pending" there in the
  first PR opened. Then edit each body once with `gh pr edit <n> --body-file` to fill the
  sibling links. With `continue`, a continued PR gets no body edit: its comment is posted
  after every PR of the run is open, so its "Related pull requests" section is filled from
  the start, and the `gh pr edit` pass covers only the PRs this run opened. Watch CI per
  PR, and comment on each issue with every PR link. `done` needs every PR green. A
  `blocked` in any repository blocks the run, and no further publication happens in any
  repository, with one exception: the sibling-link edit of a PR this run already opened
  still runs, so no PR is left saying "pending". The exception covers only PRs this run
  opened; a continued PR whose comment was not yet posted gets none. Editing the body of
  this run's own PR is inside the approval scope and publishes nothing new.
- Step 7.3.5: a CI repair review for an additional repository uses the Step 5.2 patch rule
  (Codex through `codex-lite:ask --resume`, and the repository's Claude subagent), never
  `codex-lite:review`.
- Closing references: an issue is closed only by the PR in its own repository (`Closes #n`).
  Every other PR of the run cites it as `Refs <owner>/<repo>#n`. The issue status comment
  names the PR in the issue's repository first, then the siblings.
