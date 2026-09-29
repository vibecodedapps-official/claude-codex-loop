# Changelog

## Unreleased

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
