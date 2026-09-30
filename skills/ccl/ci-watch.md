# CI watch

Read this file at Step 7.3 of `SKILL.md`. It holds Step 7.3 items 1 to 4: what CI to
read, which workflows apply, what passes, and how to poll. Item 5, CI repair, stays in
`SKILL.md`.

   1. Read these with `gh`, all readable with read access. If any read fails, CI cannot be
      verified: end in `blocked`, naming the failed read. Never treat a failed read as "nothing
      is required".
      - Required checks: `gh api repos/{owner}/{repo}/branches/<base> --jq .protection` gives
        `required_status_checks` (`contexts`, and `checks` with `app_id`). `<base>` is the
        PR's base branch (Step 0.2 records it with `continue`). Do not read the
        `/protection` endpoint, which returns 404 without admin rights.
        `gh api repos/{owner}/{repo}/rules/branches/<base>` gives the active rules, including
        organization rulesets: its `required_status_checks` rules add required checks and its
        `workflows` rules name required workflows by file path and repository. A required check
        is a name and, when set, the app that must report it.
      - The PR: `gh pr view <n> --json headRefOid,mergeable` for the head SHA and merge state,
        and `gh api repos/{owner}/{repo}/pulls/<n> --jq .merge_commit_sha` for the test merge
        commit. A missing `merge_commit_sha` is read again on the next poll.
      - Results, for the head commit and the test merge commit:
        `gh api "repos/{owner}/{repo}/commits/<sha>/check-runs?filter=latest&per_page=100"`
        (page on when `total_count` exceeds 100) and
        `gh api repos/{owner}/{repo}/commits/<sha>/status`, which gives the latest status per
        context. Only the latest result counts: the latest status per context, and the latest
        attempt of each check run within its own check suite, so same-named checks from
        different workflows are judged separately. Earlier attempts are report history only.
      - Required workflows: `gh api "repos/{owner}/{repo}/actions/runs?head_sha=<head sha>"`,
        comparing each run's `path` and repository with the rule's workflow file path and
        `repository_id`.
      Read both commits again after every push.
   2. A workflow applies to the PR when it triggers on pull requests, its `branches`,
      `branches-ignore`, `paths`, and `paths-ignore` filters match the PR's base branch and
      changed files, and its `types` filter, when present, includes the event the watched head
      commit produced: `opened` for the first watch after the PR is created, `synchronize`
      for the first watch when the run continued a branch whose PR was already open, and
      after a CI repair push to the open PR. A filter that cannot be evaluated with
      confidence counts as a match. A workflow triggered by `pull_request` (not
      `pull_request_target`) does not apply when the PR head commit's message carries a skip
      instruction: `[skip ci]`, `[ci skip]`, `[no ci]`, `[skip actions]`, `[actions skip]`, or
      a `skip-checks:true` or `skip-checks: true` trailer. The report names each workflow a
      skip instruction made not applicable. A required check stays required either way.
      Expected deferred checks are a separate set: every check Step 6 deferred to CI whose
      workflow applies, matched by job name. A deferred check whose workflow does not apply is
      named in the report as not triggered, with the filter that excluded it, unless it is
      also a required check.
   3. A result passes when it is `success`, `neutral`, or `skipped`. Any other finished
      result is a failure: `failure`, `cancelled`, `timed_out`, `action_required`, `stale`, or
      a commit status of `failure` or `error`. The gated commit is the test merge commit when
      it has any status or check run, else the head commit, because GitHub judges required
      checks on the test merge commit when it has a status. A required check is met when its
      latest result on the gated commit passes, from the required app when one is set, and,
      when the name exists both as a check run and as a status, both pass. A required app is
      verified from a check run's app. A commit status carries no app, so when a required check
      names an app and only a status carries that name, its source cannot be verified: end in
      `blocked` at once, naming the check, rather than risk `done` while GitHub rejects the
      source. A required workflow is met when its latest run for the head commit, matched by
      the rule's workflow file path and repository, passes. A match that cannot be confirmed
      counts as unmet and is named in the report.
   4. Poll at about 30 second intervals, checking the run budget each time. CI is not judged
      until 2 minutes after the push. If `mergeable` is `CONFLICTING`, `pull_request`
      workflows do not run: end in `blocked` at once, naming the conflict. CI is green when
      every required check and required workflow is met, every expected deferred check has
      passed on the head commit, every applicable workflow has reported at least one check on
      the head commit, and every latest result on either commit has finished and passed. That
      is stricter than GitHub's merge gate, on purpose: the loop publishes only fully green
      work, and the report says so when an optional check blocked it. Anything unmet or not
      yet reported is pending until the CI budget expires, then `blocked`. A failure in a
      latest result is a CI failure. CI is not applicable only when there are no required
      checks or workflows, no workflow applies, no deferred check is expected, and no result
      has appeared on either commit within 2 minutes of the push; the report says so. A result
      that appears on either commit keeps the watch open until it finishes.
