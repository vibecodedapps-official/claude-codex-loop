# Worktree run

Read this file as soon as the exception in Step 0.3 applies, before the worktree is
created. It holds the rules for a run in the detached worktree, and they apply for the
rest of the run.

   - With `continue`, a branch that is checked out in the session's checkout (`git
     worktree list --porcelain`) cannot be checked out in the worktree: end `blocked`
     naming the branch and that checkout, before the worktree is created. In a plan-only
     run, record it in `run.md` and the report instead. Step 0.2 ran the flagged-file
     check first, predicted this worktree run, left the session's files alone, and with
     consent detached the session in place when it was on the branch and fast-forwarded a
     behind local branch without checking it out, so this check passes whenever that
     consent was given; it ends here only when the branch is still checked out in the
     session, which means the question was not asked, for example in Multi-repo mode.
     Step 0.2 already fails a branch checked out in any other worktree. Otherwise Step
     3.7.2 switches to it inside the worktree.
   - Commands wrapped in `cd <checkout> && ...` are not pre-approved, and `git worktree add`
     itself may prompt. Step 0.1 already listed them, because it ran the flagged-file check
     before its statement. Record each prompt in `run.md`.
   - A detached worktree from the base commit has no installed dependencies, build output, or
     local env files. Before the baseline in Step 3.7.3, run the install step the repository's
     instruction files or lockfile name (for example `npm ci`) inside the worktree. The
     install runs under the check budget like a check (Budgets, enforcement items 2 and
     3). When none is known and a discovered check fails for that reason, record the
     check as not run in the worktree with the reason, not as a baseline failure.
   - Step 0.3 first performs the Ignoring `.ccl/` setup and allocates the run id, the work
     Step 0.5 would do first, then creates the worktree, so `.ccl/` is ignored before the
     first write. From then on the run directory exists, so a later preflight failure writes
     the report as Final report handling item 3 says. The "writes nothing" rule applies only
     when the run directory does not exist.
   - A worktree run has two absolute roots, recorded in `run.md`: `<checkout>`, the worktree,
     and `<artifacts>`, the run directory `.ccl/<run-id>/` under the original checkout.
   - Every later git command runs as `git -C <checkout> ...`. Every other command that acts
     on the tree, each repo check (Step 3.7.3, Step 4, Step 5.1, Step 6), the `git add -N`
     marking, and every `gh` call that reads the current branch, runs inside one Bash call as
     `cd <checkout> && <command>`, with every path it is given absolute. A bare `cd` that
     outlives the call is never used.
   - Step 7.2 opens the PR from `<checkout>` with the branch named explicitly (`cd
     <checkout> && gh pr create --head <branch> --body-file <artifacts>/pr-body.md ...`).
   - Every artifact path, including `diff.patch` and the `specs/ccl/<run-id>/` copy source of
     Step 7.1, is written under `<artifacts>`. Every Codex request names files by their path
     relative to the session's checkout (`.ccl/<run-id>/diff.patch`), which is where Codex
     runs. This applies to reviewer calls only. A Codex implementer call runs in
     `<checkout>` through `--cwd <checkout>`, so its request names files by absolute path
     or carries the content inline.
   - Every implementer prompt and fallback reviewer prompt names `<checkout>` as the only
     checkout to edit or read (`git -C <checkout> diff <base-commit>`). A Codex
     implementer call passes `--cwd <checkout>` as its last option. `diff.patch` is
     produced from `<checkout>` into `<artifacts>`.
   - Because codex-lite reviews the session's checkout, every diff review goes through
     `codex-lite:ask` with `diff.patch`, in a fresh `codex-lite:ask` thread that becomes the
     stage's thread.
   - Because the Claude `code-review` skill also reviews only the session's checkout, the
     Claude slot of every Step 5 round, at every tier, is the Opus subagent substitute
     that `multi-repo.md` defines for additional repositories, as Claude review contract
     item 7 in `SKILL.md` says. It reads `<artifacts>/diff.patch`, produced from
     `<checkout>`, and is continued with SendMessage in later rounds. There is no tier
     limit on a worktree run.
   - At the terminal state the worktree is kept. The report names its path and how to remove
     it (`git worktree remove <path>`).
