# Tiers, roles, and fallbacks

Read this file during Step 1, before the estimate in Step 1.5. It holds the effort tier
table, the estimate rule, the implementer choice at xhigh and max, the risk floor,
re-evaluation, and the reviewer roles with their fallbacks.

## Roles

The reviewers are keyed by reviewer, not by tier. The tier table below says which reviewer
each stage uses at each tier.

| Role | Default | Fallback when the default is unavailable |
|---|---|---|
| Orchestrator and primary reviewer | The session's current Claude model (Opus or Fable) | none, the run stops |
| Codex reviewer, `gpt-6-sol` | Skill tool, `codex-lite:ask` or `codex-lite:review`, model `gpt-6-sol` | Agent tool, model `opus` |
| Codex reviewer, `gpt-6-astra` | Skill tool, `codex-lite:ask` or `codex-lite:review`, model `gpt-6-astra` | Agent tool, model `fable`; on an error from that call, model `opus` |
| Claude reviewer (Step 5, high tier and above) | Skill tool, `code-review`, at the tier's level | none; if the skill is not listed when the stage starts, the run ends in `blocked` |
| Implementer, low, medium, and high tier | Agent tool model `sonnet` | none, the run stops |
| Implementer, xhigh and max tier | Agent tool model `sonnet`, or `opus` when the Opus criteria apply to the slice | on a tool error from an `opus` call, `sonnet`, and the error is recorded |

Rules for roles:

- Codex model ids are always the full id, `gpt-6-sol` or `gpt-6-astra`. A bare id such as
  `sol` or `astra` fails on a ChatGPT account. Pass the full id, and `--timeout` from the
  Codex budget, on every Codex call, including `--resume` follow-ups.
- Codex is reached only through the Skill tool, with `codex-lite:ask` for plans and
  questions and `codex-lite:review` for diffs. Never run the `codex` CLI directly.
- For a `gpt-6-astra` stage at any tier, the fallback tries the `fable` model on the Agent
  tool first. If that call returns an error, use `opus` and record the error. Do not detect
  the session's model.
- The Claude reviewer is the built-in `code-review` skill, called through the Skill tool
  with the level the tier table names as its only argument before any target, never
  `--comment` and never `--fix`. It is a fixed slot beside the Codex slot at high tier and
  above: it is not a fallback for the Codex reviewer, and nothing falls back to it or
  replaces it. `--no-codex` does not touch it. Its availability is checked only when a stage
  that needs it starts, so low tier, medium tier, and plan-only runs do not need it. The
  Reviewer contract in `SKILL.md` gives the call shape and the budget.
- An implementer call at model `opus`, through the Agent tool or inside a Workflow, whose
  tool call itself returns an error is rerun with the same prompt at `sonnet`. The slice's
  effective model becomes `sonnet`. Log the error and the swap, and name it in the report
  as an implementer swap. Do not stop the run. This does not cover a permission denial
  (Approval scope, carve-out 3), a call that runs past its subagent timeout (a budget
  expiry), or any reviewer call, whose fallbacks stay as the table lists.
- A fallback swaps one reviewer or one implementer. It never removes a stage. The tier is
  set by the task's risk and does not change because a reviewer is unavailable.
- A fallback reviewer gets the same request text, the same files, and the same required
  reply shape as the Codex reviewer it replaces.
- Fall back to the Claude subagent when `--no-codex` is set, when Codex was found
  unavailable in Step 0.6, or when a Codex call returns `failed` or no status line twice
  in a row. A `refused` status is not retried and is not swapped: it ends the run in
  `blocked` with the message. A `timeout` status is a budget expiry and ends the run in
  `blocked` with the Codex budget named.
- Write every swap to the run log with the stage, the reason, and the fallback model. Every
  swap is named in the final report.
- If no reviewer is available for a required stage, the run ends in `blocked`. If the
  orchestrator is unavailable, or an implementer call at `sonnet` errors, the run stops.

## Effort tiers

| Step | Low | Medium | High | xhigh | Max |
|---|---|---|---|---|---|
| 1 Review and verify | yes | yes | yes | yes | yes |
| 2 Plan | orchestrator drafts, one or more slices | orchestrator drafts, one or more slices | orchestrator drafts, one or more slices | orchestrator drafts, one or more slices | orchestrator drafts, one or more slices |
| 3 Plan review and converge | Codex `gpt-6-sol`, converge | Codex `gpt-6-sol`, converge | Codex `gpt-6-astra` with a trigger, else `gpt-6-sol`, converge | Codex `gpt-6-astra`, converge | Codex `gpt-6-astra`, converge |
| 4 Implement | Sonnet, one per slice, orchestrator reviews | Sonnet, one per slice, orchestrator reviews | Sonnet, one per slice, orchestrator reviews | Sonnet or Opus per slice, orchestrator reviews | Sonnet or Opus per slice, orchestrator reviews |
| 5 Final review | skipped | Codex `gpt-6-sol` | Codex `gpt-6-sol` and Claude `code-review medium` | Codex `gpt-6-astra` with a trigger, else `gpt-6-sol`, and Claude `code-review high` | Codex `gpt-6-astra` and Claude `code-review xhigh` |
| 6 Checks | yes | yes | yes | yes | yes |
| 7 Publish | yes | yes | yes | yes | yes |

"With a trigger" means a risk floor trigger exists for the change: it adds, alters, or
removes an item in the risk floor list, directly or through shared code. It does not mean
the floor raised the tier. Only two cells depend on it. For the high tier plan review, the
trigger is judged from the Step 1.5 floor check: a floored high run gets `gpt-6-astra`, a
high run that is only cross-cutting gets `gpt-6-sol`. For the xhigh final review, a
trigger counts when it was present at the estimate or is present in the diff after Step
4: an xhigh run forced with `--effort xhigh` on a floored task, or whose diff gained a
trigger, gets `gpt-6-astra` at Step 5 while staying xhigh. Every other cell is fixed by
the tier alone; the high tier final review is `gpt-6-sol` whether or not a trigger exists.

Low tier skips all of Step 5, including 5.1; Step 6 then runs the full check set. Every
tier reviews the plan in Step 3. At high tier and above a Step 5 round is both reviewers,
Codex and Claude, over the same diff. Every implementer call keeps the per-call subagent
timeout, Step 4 keeps its cap of 3 rounds per slice, and the run budget still bounds the
whole run. A plan with three slices has three independent round caps, and an implementer
swap adds a call to that slice. `--plan-only` stops at Step 3.6 at every tier.

## Estimate rule

Apply after Step 1. Low, medium, and high are sized from the behavior the change has, not
from how many issues there are. xhigh and max are sized from how many areas that share no
file the change spans, on top of that.

- Low: one file or one function, a clear fix, and none of the risk floor triggers.
- Medium: several files in one area, or one issue with tests, or any doc restructure.
- High: a cross-cutting change inside one deliverable, or any risk floor trigger.
- xhigh: one change whose scope spans several areas of the code that share no file. This
  sets review depth. Slice count is set by the plan at every tier.
- Max: an xhigh-shaped change that also has a risk floor trigger.

Bundling issues does not by itself raise the tier; estimate the bundle as one change. Two
issues that each touch one file in one area are still medium. A bundle is xhigh only when
the change it describes, taken as one change, spans several areas with no shared file.

`--effort low|medium|high|xhigh|max` skips the estimate and forces that tier, subject to
the risk floor below.

Record the estimate, the reason, any floor applied, and any re-evaluation in the run log
and in the final report.

## Implementer choice at xhigh and max

At low, medium, and high tier every implementer is `sonnet`. At xhigh and max, choose the
implementer model per slice in Step 2. Choose `opus` for a slice when any of these hold;
otherwise choose `sonnet`:

- the slice carries a risk floor trigger: its change adds, alters, or removes an item in
  the risk floor list
- the slice owns more than eight files, counting files it creates
- the slice adds a new module, type, interface, or rule section that another file in the
  slice, or in another slice, calls, implements, or cites

Record the choice in the plan next to the slice, with the criterion that applied, or
"none" for `sonnet`. Log it in the run log when the slice's agent starts, and list it in
the final report per slice with the reason. The plan reviewer may object to a choice; the
objection is handled like any other.

The slice's effective model is the chosen model, or `sonnet` after an implementer swap (see
Rules for roles). Every later call for that slice, in Step 4.3, Step 5.3, and CI repair,
whether the agent is continued or fresh, including a fresh agent replacing one that ran
inside a Workflow, uses the effective model. The single fix after the Step 4 cap stays the
orchestrator's.

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
than planned does not move the run to xhigh or max. The check has two outputs: whether
the tier rises, and which Step 5 reviewers the run gets.

- If a trigger now exists and the run is below high tier, the run rises to high tier. Log
  the rise and the reason. A run already at high, xhigh, or max tier keeps its tier,
  whatever the diff contains.
- Resolve the Step 5 reviewers from the tier table after this check. Only the xhigh cell
  depends on the diff: an xhigh run gets Codex `gpt-6-astra` when a trigger was present at
  the estimate or is present in the diff, else `gpt-6-sol`, beside Claude `code-review
  high`, and stays xhigh. Every other tier's cell stands as the table gives it. A medium
  run that rose to high gets the high cell: Codex `gpt-6-sol` and Claude `code-review
  medium`.
- Complete Step 5 with those reviewers before Step 6. It stays inside the round caps in the
  budgets. Plan review at Step 3 is not repeated after Step 4, so a run that rose keeps
  the `gpt-6-sol` plan review it already had.
- The tier never falls after Step 4 because the diff turned out smaller than planned, and
  the Step 5 reviewers never weaken: at xhigh, a trigger present at the estimate counts
  even if the diff no longer shows it.
- The estimate, its reason, any floor applied, the re-evaluation, and the Step 5 reviewers
  it resolved go in the final report.
