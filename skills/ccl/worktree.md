# Worktree run

Read this file as soon as the exception in Step 0.3 applies, before the worktree is
created. It holds the rules for a run in the detached worktree, and they apply for the
rest of the run.

   - With `continue`, a branch that is checked out in the session's checkout (`git
     worktree list --porcelain`) cannot be checked out in the worktree: end `blocked`
     naming the branch and that checkout, before the worktree is created. Step 0.2 already
     fails a branch checked out in any other worktree. Otherwise Step 3.7.2 switches to
     it inside the worktree.
   - Commands wrapped in `cd <checkout> && ...` are not pre-approved, and `git worktree add`
     itself may prompt. Step 0.1 already listed them, because it ran the flagged-file check
     before its statement. Record each prompt in `run.md`.
   - A detached worktree from the base commit has no installed dependencies, build output, or
     local env files. Before the baseline in Step 3.7.3, run the install step the repository's
     instruction files or lockfile name (for example `npm ci`) inside the worktree. When none
     is known and a discovered check fails for that reason, record the check as not run in
     the worktree with the reason, not as a baseline failure.
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
     runs.
   - Every implementer prompt and fallback reviewer prompt names `<checkout>` as the only
     checkout to edit or read (`git -C <checkout> diff <base-commit>`). `diff.patch` is
     produced from `<checkout>` into `<artifacts>`.
   - Because codex-lite reviews the session's checkout, every diff review goes through
     `codex-lite:ask` with `diff.patch`, in a fresh `codex-lite:ask` thread that becomes the
     stage's thread.
   - Because the Claude `code-review` skill also reviews only the session's checkout, a
     worktree run is allowed only below high tier. When Step 1.5 sets high or above, or Step
     4.5 raises the run to high, the run ends in `blocked` naming the skip-worktree
     state and the tier, before anything is implemented in the first case and before Step 5
     in the second.
   - At the terminal state the worktree is kept. The report names its path and how to remove
     it (`git worktree remove <path>`).
