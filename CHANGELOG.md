# Changelog

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

### Known limits

- Default permission mode prompts at every Codex call. Unattended runs need auto mode,
  or `--no-codex` plus allow rules for the `git` and `gh` writes.
- No marketplace listing yet. Install from a local clone with `claude --plugin-dir`.
- Some mechanics are unverified until the acceptance checks are run by hand: the
  command-to-skill handoff, whether Codex reviews files marked with `git add -N`,
  whether a command can pre-approve another plugin's Bash call, and the Fable model
  override on the Agent tool.

### Deferred to 0.2

- Repair mode, `/ccl:pr <n>`, which fixes an existing pull request.
- `--merge`, which merges after CI is green and cleans up the branch.
- `--worktree`, which runs the loop in a new worktree.
