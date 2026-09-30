# Changelog

## 0.6.0 - 2026-09-30

### Added

- `--continue <branch>` on `/ccl:run` and `/ccl:plan`. The run takes the remote branch
  head as the base, switches to the branch instead of creating one, pushes to it without
  force, and comments on its open PR instead of opening a second one, or opens a PR when
  none exists. PR references stay rejected as inputs. The value is rejected when it starts
  with `-`, has a character other than letters, digits, `.`, `_`, `/`, and `-`, or fails
  `git check-ref-format --branch`. A branch checked out in another worktree fails
  preflight. `HEAD` of the session's checkout must be at the base commit, on the branch or
  detached at it, or preflight fails with the command to run, which is `git merge
  --ff-only <remote>/<branch>`, `git switch <branch>`, or `git switch --detach
  <remote>/<branch>`, by where the session is. The default branch is rejected as the
  value. A plan-only run records a differing local branch, a branch checked out in another
  worktree, and the pull request cases instead of failing, but it still needs `HEAD` at
  the remote head, for example detached with `git switch --detach <remote>/<branch>`, and
  then records the differing local branch. A `--no-publish` run also records the closed,
  merged, and several-open pull request cases instead of failing. A repository whose
  remote lacks the branch creates it under the same name in Multi-repo mode. A PR opened
  for a continued branch says how many earlier commits the run did not review. The pull
  request list reads open PRs first, up to 100, and the rest only when none from this
  repository is open; a list that may be cut off by forks with the same branch name is a
  pull request failure. After a `--confirm-plan` yes and before the first push the PRs
  are read again, and a PR retargeted to another base branch ends in `blocked`. A
  `prepared` run with an open PR gives `gh pr comment <n> --body-file <absolute path>` in
  place of `gh pr create`. With `--no-publish` and an incomplete list, several open PRs,
  or only closed or merged ones, it names them and gives neither command. (#16)
- `--confirm-plan` and Step 3.5. Once the plan is final and reverified, the run asks once.
  A clear yes continues, a requested change is one more Step 3 round, so an open blocking
  objection at the cap ends in `blocked`, and anything else ends in plan-only. The wait is
  left out of the run budget. After a yes, the clean-tree check reruns before Step 3.7.2,
  with the local-branch, worktree, and `HEAD` checks under `--continue`, and so does the
  skip-worktree comparison, where a hidden edit not recorded at Step 0.3 is a failure. A
  failure ends in `blocked`. A requested change with no round left ends in `stopped`, and the
  report gives the change. (#17)
- A Step 1 check for writable checkouts named in the task but not passed with `--repo`.
  The run ends in `blocked` with the rerun command instead of adopting Multi-repo mode
  from prose. The check covers any writable checkout not listed with `--repo`, and the
  rerun command adds one `--repo` for each missing checkout. (#18)
- A rule for an additional repository with skip-worktree files that differ from `HEAD`.
  The run continues, and the report names the files as the local state its checks ran
  against. No slice may edit such a path, because the edit would be invisible to review
  and the commit: a plan that needs one ends in `blocked` at Step 2. (#18)

### Changed

- The worktree run, Multi-repo mode, and CI watch rules moved from `SKILL.md` to
  `worktree.md`, `multi-repo.md`, and `ci-watch.md`, read only when the run takes that
  path. The move commit changed no rule, only pointers and cross-references. The later
  0.6.0 rule changes in those files are the ones listed in this section. The move cut the
  lines a default single-repository GitHub run reads through Step 6 from 1,309 to 1,115.
  (#20)

### Fixed

- Every `code-review` pass now targets `<base-commit>...HEAD`. A bare base commit made
  it review only that commit and skip the uncommitted task. Verified live on 2026-09-30
  in the Step 5 and CI repair states. (#15)
- Every remote branch check queries the exact ref `refs/heads/<branch>`. A bare name
  matches any ref whose path ends in it, which could report a missing branch as present.
  This includes the new-branch collision check in Step 3.7.2. (#16)
- In Multi-repo mode each repository's PR body is `.ccl/<run-id>/pr-body-<slug>.md` in
  the primary, passed by absolute path to every `gh` body call for an additional
  repository. The relative path that `gh pr create` used before would not resolve from
  that repository's checkout. (#18)
- Time logging runs `date` only at step headings and around timed calls, in one UTC
  format. Every logged time is copied from the `date` output, not recalled or derived.
  (#19) Elapsed time is the Step 0 start to the latest recorded time, less each Step 3.5
  wait. The worktree install step is a timed call under the check budget.
- The CI watch reads the PR again at each poll and ends in `blocked` when its base branch
  differs from the one the run recorded, because the required checks it read belong to
  the old base branch.

## 0.5.1 - 2026-09-29

### Changed

- The `gpt-6-sol` reviewer slot now uses `gpt-6.1-sol`, released 2026-09-29. Every cell
  that named `gpt-6-sol` keeps its place in the tier table; only the model id changes.
  `gpt-6-astra` cells and the Opus fallback are unchanged.

## 0.5.0 - 2026-09-29

### Changed

- Codex availability no longer depends on the session's skill list. It is decided from
  `codex --version` and the installed codex-lite version, 0.7.0 or later. A Skill call for
  `codex-lite:ask` or `codex-lite:review` that errors because the skill is not listed
  counts as a `failed` call: retry once, then swap, with the reason recorded. (#7)
- Implementers match the repository's line endings for every new file, and Step 4.3
  checks new files with `git ls-files --eol` before review. (#8)
- Independent slices run in parallel either as one Workflow or as parallel Agent calls
  in one message. The orchestrator picks and logs the choice in `run.md`. Agent calls
  are the default when review rounds are expected, because they can be continued. (#9)
- The run budget default is by tier: 120 minutes at low and medium, 240 at high, 360 at
  xhigh and max. `--run-budget <minutes>`, else `.ccl.json` `timeouts.run`, overrides it,
  and an explicit instruction in the session can replace it. The report names the budget
  in force and its source. (#10)
- A dropped call, one that returns no result and no explicit denial, is not a denial. A
  dropped read-only call is retried once, serially. A dropped write is checked before any
  retry. A clean tree with skip-worktree or assume-unchanged files that differ from
  `HEAD` runs in a detached worktree beside the checkout, below high tier and
  not in Multi-repo mode. This is a narrow use of the deferred `--worktree` feature, not
  the feature. (#11)

### Added

- Host detection in Step 0. On a host other than GitHub the run accepts only file and
  text inputs, runs Steps 0 to 6, and ends in `prepared` with a handoff to the host's own
  tooling. A full Azure DevOps path is out of scope. (#4)
- Multi-repo mode: a repeatable `--repo <path>` flag names additional writable
  checkouts. Each repository gets its own base commit, branch, baseline, checks, and PR,
  with sibling links in each PR body. Every repository must be on the same host. (#5)
- The `prepared` terminal state, and the `--no-publish` flag on `/ccl:run`. A run whose
  Step 7 is withheld before any push, by the flag, by a non-GitHub host, or by a denied
  Step 7 action, ends in `prepared` with the commands to publish. (#6)

## 0.4.0 - 2026-09-29

### Changed

- Every tier reviews the plan with Codex. Low tier no longer skips Step 3.
- Only low tier skips the final review. Medium now gets a `gpt-6-sol` diff review.
- At high tier and above the built-in `code-review` skill reviews the diff beside Codex,
  at `medium` for high, `high` for xhigh, and `xhigh` for max. It is a fixed slot: never
  swapped, untouched by `--no-codex`, and a run that needs it and cannot find it ends in
  `blocked`. Low, medium, and plan-only runs do not need it.
- Two cells follow the risk trigger: the high tier plan review is `gpt-6-astra` when the
  change has a risk floor trigger at the estimate, else `gpt-6-sol`; the xhigh final
  review is `gpt-6-astra` when a trigger was present at the estimate or is in the diff
  after Step 4, else `gpt-6-sol`. Xhigh plan review is now always `gpt-6-astra`. The high
  tier final review stays `gpt-6-sol`.
- A Step 5 round at high tier and above is both passes over the same diff, under one
  shared cap of 3. Findings from both are merged and fixed in one batch. Step 5 no longer
  has a post-cap orchestrator fix, and the third round fixes nothing: a blocking finding
  there ends in `blocked` and a non-blocking one is deferred.
- Every Claude review pass is given the base commit as its target, so it reviews the same
  base-to-working-tree diff Codex does. Without a target the skill picks its own range and
  can include unrelated commits when local `main` is behind the fetched base.
- CI repair and Step 6 repair at medium tier and above go through a Step 5 round with every
  reviewer the stage has. Low tier keeps the orchestrator's own review.
- The reviewer fallback follows the Codex model: `gpt-6-sol` to Opus, `gpt-6-astra` to
  Fable then Opus, at any tier.
- Decisions Part 7 records the reasons. Part 3 items 9 and 11 and Part 5 item 5 are
  superseded; Part 2 item 10, Part 4 item 9, and Part 5 items 3 and 7 are qualified.
- Acceptance items 4, 27, 29, 36, 38, 41, 42, and 44 change, and items 55 to 61 are new.

## 0.3.0 - 2026-09-28

### Changed

- Slice count is now a plan property at every tier. A plan has one or more slices that
  share no file: independent slices run in parallel in one Workflow, dependent slices run
  in order. `xhigh` and `max` are still sized by how many areas the change spans, but that
  now sets review depth only.
- At `xhigh` and `max` the implementer for each slice is Sonnet or Opus, chosen by the
  written criteria in `tiers.md`. If an Opus call returns a tool error, the slice falls
  back to Sonnet and the swap is recorded. Low, medium, and high still use Sonnet.
- Budget consequence: each slice keeps its own cap of 3 rounds and the per-call subagent
  timeout, and the run budget still bounds the whole run. A high plan with three slices
  therefore has three independent round caps where 0.2.0 had one.
- Decision Part 5 item 4 is superseded by Part 6 in `docs/decisions.md`.

## 0.2.0 - 2026-09-28

### Changed

- Five effort tiers, `low|medium|high|xhigh|max`, replace the three. Plan review runs at
  medium and above (`gpt-6-astra` at max, `gpt-6-sol` otherwise). The final review runs at
  high and above (same models). Low, medium, and high plans have one slice built by one
  Sonnet agent; xhigh and max plans may have several, one Sonnet agent per slice, in
  parallel when independent. Compared with 0.1.0, `--effort high` now gets `gpt-6-sol`
  reviews instead of `gpt-6-astra` and a single implementer, and `--effort medium` no
  longer gets a final review; the old high is closest to the new `max`.
- The Codex round review during implementation is removed at every tier. The orchestrator
  is the only reviewer during implementation.
- The risk floor now targets high, the middle tier. `--effort low` and `--effort medium`
  are refused on a floored task; `xhigh` and `max` are above the floor and honored.
- Medium tier no longer has a final review. It keeps the plan review, and Step 6 still
  runs the full check set.
- The estimate rule has five buckets. Low, medium, and high are sized from behavioral
  risk; xhigh and max from how many areas that share no file the change spans.
- After implementation only the risk floor is applied to the diff again. The estimate
  rule is not, so a run never rises above high after Step 4.

## 0.1.0 - 2026-09-28

First release.

### Added

- Plugin `ccl`, prompt-only: markdown and one JSON manifest, no hooks and no scripts.
- Commands `/ccl:run` and `/ccl:plan`. `/ccl:plan` is `/ccl:run --plan-only`. Both run
  build mode and share one orchestrator skill.
- Inputs: same-repo issue URLs or `#n` numbers (bundled into one PR), an ad-hoc
  description, and a file of notes. Pull request references are rejected.
- Flags: `--effort low|medium|high`, `--plan-only`, `--no-codex`, `--branch <name>`.
- Effort tiers with an estimate rule, a risk floor for auth, permission, schema,
  migration, row-level security, data access, and public API changes, and a tier
  re-evaluation against the actual diff.
- Steps 0 to 7: preflight, review and verify, plan, plan review, execution setup,
  implement, final review, checks, publish. Publish commits, pushes, opens one PR,
  watches CI with up to 3 repair cycles, and comments on each source issue.
- Codex review through `codex-lite` 0.7.0 or later, with full model ids `gpt-6-sol` and
  `gpt-6-astra`, a `--timeout` on every call, and explicit thread ids on follow-ups.
- Claude fallbacks for every reviewer stage, named in the report. A fallback never
  removes a stage.
- Approval scope: what the run does without asking, what it always asks before, and a
  rule that repo and user ask-first rules win.
- Terminal states `done`, `plan-only`, `blocked`, and `stopped`, each with a written
  report, except that a failure before the run directory exists prints the report and
  writes nothing.
- Round caps of 3, per-call time budgets, and a 4 hour run-wide budget, overridable in
  `.ccl.json`.
- `.ccl.json` with `commit`, `checks`, and `timeouts`.
- Templates for the final report and the PR body.
- `docs/decisions.md` (why each rule exists) and `docs/acceptance.md` (hand-run checks).
- `.claude-plugin/marketplace.json`, so the repo can be added as a marketplace and the
  plugin installed with `/plugin install ccl@vibecodedapps-claude-codex-loop`.

### Known limits

- Default permission mode prompts at every Codex call. Unattended runs need auto mode,
  or `--no-codex` plus allow rules for the `git` and `gh` writes.
- Some mechanics are unverified until the acceptance checks are run by hand: the
  command-to-skill handoff, whether Codex reviews files marked with `git add -N`,
  whether a command can pre-approve another plugin's Bash call, and the Fable model
  override on the Agent tool.

### Deferred to 0.2

- Repair mode, `/ccl:pr <n>`, which fixes an existing pull request.
- `--merge`, which merges after CI is green and cleans up the branch.
- `--worktree`, which runs the loop in a new worktree.
