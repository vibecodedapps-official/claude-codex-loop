# Tiers, roles, and fallbacks

Read this file during Step 1, before the estimate in Step 1.5. It holds the effort tier table, the
estimate rule, the risk floor, re-evaluation, and the reviewer roles with their fallbacks.

## Roles

| Role | Default | Fallback when the default is unavailable |
|---|---|---|
| Orchestrator and primary reviewer | The session's current Claude model (Opus or Fable) | none, the run stops |
| Plan reviewer, medium tier | Codex `gpt-6-sol` | Agent tool, model `opus` |
| Plan reviewer, high tier | Codex `gpt-6-astra` | Agent tool, model `fable`; on an error from that call, model `opus` |
| Implementer | Claude Sonnet subagents, Agent tool model `sonnet` | none, the run stops |
| Round reviewer (Step 4.4), high tier | Codex `gpt-6-sol` | Agent tool, model `opus` |
| Final reviewer (Step 5), medium tier | Codex `gpt-6-sol` | Agent tool, model `opus` |
| Final reviewer (Step 5), high tier | Codex `gpt-6-astra` | Agent tool, model `fable`; on an error from that call, model `opus` |

Rules for roles:

- Codex model ids are always the full id, `gpt-6-sol` or `gpt-6-astra`. A bare id such as
  `sol` or `astra` fails on a ChatGPT account. Pass the full id, and `--timeout` from the
  Codex budget, on every Codex call, including `--resume` follow-ups.
- Codex is reached only through the Skill tool, with `codex-lite:ask` for plans and
  questions and `codex-lite:review` for diffs. Never run the `codex` CLI directly.
- For the astra stages, try the `fable` model on the Agent tool first. If that call
  returns an error, use `opus` and record the error. Do not detect the session's model.
- A fallback swaps one reviewer. It never removes a stage. The tier is set by the task's
  risk and does not change because a reviewer is unavailable.
- A fallback reviewer gets the same request text, the same files, and the same required
  reply shape as the Codex reviewer it replaces.
- Fall back to the Claude reviewer when `--no-codex` is set, when Codex was found
  unavailable in Step 0.6, or when a Codex call returns `failed` or no status line twice
  in a row. A `refused` status is not retried and is not swapped: it ends the run in
  `blocked` with the message. A `timeout` status is a budget expiry and ends the run in
  `blocked` with the Codex budget named.
- Write every swap to the run log with the stage, the reason, and the fallback model. Every
  swap is named in the final report.
- If no reviewer is available for a required stage, the run ends in `blocked`. If the
  orchestrator or the implementer is unavailable, the run stops.

## Effort tiers

| Step | Low | Medium | High |
|---|---|---|---|
| 1 Review and verify | yes | yes | yes |
| 2 Plan | orchestrator drafts | orchestrator drafts | orchestrator drafts |
| 3 Plan review and converge | skipped | Codex sol, converge | Codex astra, converge |
| 4 Implement | one Sonnet agent, orchestrator reviews | Sonnet, one per slice, orchestrator reviews | Sonnet, one per slice, orchestrator and Codex sol review |
| 5 Final review | skipped | Codex sol | Codex astra |
| 6 Checks | yes | yes | yes |
| 7 Publish | yes | yes | yes |

Low tier skips Step 3 and all of Step 5, including 5.1. Step 6 then runs the full check
set. `--plan-only` stops at Step 3.6 at every tier, including low.

## Estimate rule

Apply after Step 1, from the behavior the change has, not from how many issues there are.

- Low: one file or one function, a clear fix, and none of the risk floor triggers.
- Medium: several files in one area, or one issue with tests, or any doc restructure.
- High: a cross-cutting change, or any risk floor trigger.

Bundling issues does not by itself raise the tier. Estimate the bundle as one change.

`--effort low|medium|high` skips the estimate and forces that tier, subject to the risk
floor below.

Record the estimate, the reason, any floor applied, and any re-evaluation in the run log
and in the final report.

## Risk floor

A change is high tier if it adds, alters, or removes any of these, whether directly or
through shared code or configuration that they depend on:

- an auth check
- a permission rule
- a schema or a migration
- a row-level security policy
- a data access path
- a public API's signature or behavior

The triggers are about behavior. Editing a file in one of these areas without changing
such behavior does not trigger the floor. Examples are a typo fix in a migrations README,
a comment change in an auth module, or a rename that changes no signature. This is the
incidental-edit exception. When it applies, the final report says why the floor did not.

`--effort` cannot lower a task below the floor. If the user passes a lower `--effort` for
a task the floor covers, refuse the request, state the reason (which trigger applies), and
continue the run at high tier. Do not stop the run.

## Re-evaluation after Step 4

After Step 4, evaluate the tier again against the actual diff, using the same estimate rule
and risk floor.

- If the floor now applies and the run is below high tier, the run rises to high tier. Log
  the rise and the reason.
- Complete the reviews high tier requires before Step 6: the Step 4.4 Codex sol round
  review of the integrated work, and the Step 5 final review with the astra reviewer and its
  fallbacks. Each stays inside the round caps in the budgets. Plan review at Step 3 is not
  repeated for a rise that happens after Step 4.
- The tier never falls after Step 4 because the diff turned out smaller than planned.
- The estimate, its reason, any floor applied, and the re-evaluation go in the final report.
