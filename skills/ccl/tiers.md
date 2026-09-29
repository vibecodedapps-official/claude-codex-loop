# Tiers, roles, and fallbacks

Read this file during Step 1, before the estimate in Step 1.5. It holds the effort tier table, the
estimate rule, the risk floor, re-evaluation, and the reviewer roles with their fallbacks.

## Roles

| Role | Default | Fallback when the default is unavailable |
|---|---|---|
| Orchestrator and primary reviewer | The session's current Claude model (Opus or Fable) | none, the run stops |
| Plan reviewer, medium, high, and xhigh tier | Codex `gpt-6-sol` | Agent tool, model `opus` |
| Plan reviewer, max tier | Codex `gpt-6-astra` | Agent tool, model `fable`; on an error from that call, model `opus` |
| Implementer | Claude Sonnet subagents, Agent tool model `sonnet` | none, the run stops |
| Final reviewer (Step 5), high and xhigh tier | Codex `gpt-6-sol` | Agent tool, model `opus` |
| Final reviewer (Step 5), max tier | Codex `gpt-6-astra` | Agent tool, model `fable`; on an error from that call, model `opus` |

Rules for roles:

- Codex model ids are always the full id, `gpt-6-sol` or `gpt-6-astra`. A bare id such as
  `sol` or `astra` fails on a ChatGPT account. Pass the full id, and `--timeout` from the
  Codex budget, on every Codex call, including `--resume` follow-ups.
- Codex is reached only through the Skill tool, with `codex-lite:ask` for plans and
  questions and `codex-lite:review` for diffs. Never run the `codex` CLI directly.
- For the max tier stages (`gpt-6-astra`), try the `fable` model on the Agent tool first.
  If that call returns an error, use `opus` and record the error. Do not detect the
  session's model.
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

| Step | Low | Medium | High | xhigh | Max |
|---|---|---|---|---|---|
| 1 Review and verify | yes | yes | yes | yes | yes |
| 2 Plan | orchestrator drafts, one slice | orchestrator drafts, one slice | orchestrator drafts, one slice | orchestrator drafts, one or more slices | orchestrator drafts, one or more slices |
| 3 Plan review and converge | skipped | Codex `gpt-6-sol`, converge | Codex `gpt-6-sol`, converge | Codex `gpt-6-sol`, converge | Codex `gpt-6-astra`, converge |
| 4 Implement | one Sonnet agent, orchestrator reviews | one Sonnet agent, orchestrator reviews | one Sonnet agent, orchestrator reviews | Sonnet, one per slice, orchestrator reviews | Sonnet, one per slice, orchestrator reviews |
| 5 Final review | skipped | skipped | Codex `gpt-6-sol` | Codex `gpt-6-sol` | Codex `gpt-6-astra` |
| 6 Checks | yes | yes | yes | yes | yes |
| 7 Publish | yes | yes | yes | yes | yes |

Low tier skips Step 3. Low and medium tier skip all of Step 5, including 5.1; Step 6 then
runs the full check set. Low, medium, and high plans have one slice. `--plan-only` stops at
Step 3.6 at every tier.

## Estimate rule

Apply after Step 1. Low, medium, and high are sized from the behavior the change has, not
from how many issues there are. xhigh and max are sized from how many areas that share no
file the change spans, on top of that.

- Low: one file or one function, a clear fix, and none of the risk floor triggers.
- Medium: several files in one area, or one issue with tests, or any doc restructure.
- High: a cross-cutting change inside one deliverable, or any risk floor trigger.
- xhigh: one change whose scope spans several areas of the code that share no file, so the
  plan splits it into two or more slices with disjoint files, whether they can run in
  parallel or must run in order. The split is a property of the change's scope, not of
  how many inputs describe it.
- Max: an xhigh-shaped change that also has a risk floor trigger.

Bundling issues does not by itself raise the tier; estimate the bundle as one change. Two
issues that each touch one file in one area are still medium. A bundle is xhigh only when
the change it describes, taken as one change, spans several areas with no shared file.

`--effort low|medium|high|xhigh|max` skips the estimate and forces that tier, subject to
the risk floor below.

Record the estimate, the reason, any floor applied, and any re-evaluation in the run log
and in the final report.

## Risk floor

A change is at least high tier if it adds, alters, or removes any of these, whether
directly or through shared code or configuration that they depend on:

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

The floor is `high`. `--effort` cannot lower a task below it. If the user passes
`--effort low` or `--effort medium` for a task the floor covers, refuse the request, state
the reason (which trigger applies), and continue the run at high tier. Do not stop the
run. `--effort xhigh` or `--effort max` is above the floor and is honored.

## Re-evaluation after Step 4

After Step 4, apply the risk floor to the actual diff. This is a floor check only: the
estimate rule is not applied again, so a diff that turned out larger or more independent
than planned does not move the run to xhigh or max.

- If the floor now applies and the run is below high tier, the run rises to high tier. Log
  the rise and the reason. A run already at high, xhigh, or max tier is unchanged by the
  floor, whatever the diff contains.
- Complete the reviews high tier requires before Step 6: the Step 5 final review with the
  high tier reviewer and its fallbacks. It stays inside the round caps in the budgets. Plan
  review at Step 3 is not repeated for a rise that happens after Step 4.
- The tier never falls after Step 4 because the diff turned out smaller than planned.
- The estimate, its reason, any floor applied, and the re-evaluation go in the final report.
