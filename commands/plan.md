---
description: "Plan one unit of work with the ccl loop and stop once the plan is final, changing no code. Use when the user asks to plan with ccl, or types /ccl:plan. Inputs are issue URLs or #n numbers of this repo, file paths, and a quoted description. Flags are --effort low|medium|high|xhigh|max, --no-codex, --branch <name>, --continue <branch>, --run-budget <minutes>, and --repo <path>. Pull request references are rejected. To also build and publish, use /ccl:run."
argument-hint: '<#n | issue URL | file path | "description">... [--effort low|medium|high|xhigh|max] [--no-codex] [--branch <name>] [--continue <branch>] [--run-budget <minutes>] [--repo <path>]...'
allowed-tools: Bash(git status:*), Bash(git rev-parse:*), Bash(git remote:*), Bash(gh repo view:*), Bash(gh issue view:*), Bash(gh pr view:*), Bash(git ls-remote *), Bash(git check-ref-format *), Bash(git -C * remote -v), Bash(git -C * rev-parse *), Read, Skill
---

You are a thin forwarder for the ccl orchestrator, in plan-only mode. Do the steps below in order.

1. Parse the arguments shown between the markers below. They are what the user typed after the command. Split them into inputs and flags with these rules.

<user-text>
"$ARGUMENTS"
</user-text>

   - A flag is a token that starts with `--`. Accepted flags: `--effort`, `--no-codex`, `--branch`, `--continue`, `--run-budget`, `--repo`. `--effort` takes exactly one value, one of `low`, `medium`, `high`, `xhigh`, `max`; without it, effort is `auto`. `--branch` takes exactly one value, a branch name; without it, branch is `default`. `--continue` takes exactly one value, the name of an existing branch on the remote; without it, continue is `none`. Reject a `--continue` value that starts with `-`, that contains `@{`, or that has any character other than letters, digits, `.`, `_`, `/`, and `-`; step 2 validates the rest of the value. Reject `--continue` together with `--branch`. Reject any other flag, a missing flag value, and any `--effort` value other than `low`, `medium`, `high`, `xhigh`, `max`. `--no-codex` takes no value. `--run-budget` takes exactly one value, a positive integer number of minutes; without it, run-budget is `default`. Reject any other value. `--repo` takes exactly one value, a path, and may be repeated. Without `--repo`, repos is `none`. `--plan-only` is not accepted here because this command always plans only; reject it as redundant and point to `/ccl:run --plan-only`. `--no-publish` is not accepted here either, because this command never publishes; reject it as redundant and point to `/ccl:run --no-publish`. `--confirm-plan` is not accepted here either, because this command never implements and there is no plan to approve before implementation; reject it as not applicable and point to `/ccl:run --confirm-plan`.
   - An input token that is an issue URL (`https://<host>/<owner>/<repo>/issues/<n>`, where `<host>` is `github.com` or the GitHub host that `gh repo view` resolves for the current checkout, as with GitHub Enterprise) or `#<n>` is an issue. Several issues are allowed. With `--repo`, a bare `#<n>` always names an issue of the current checkout, the primary. An issue of a `--repo` checkout must be given as a full URL.
   - A pull request URL (`https://<host>/<owner>/<repo>/pull/<n>`, with the same `<host>` rule) is a pull request. A `#<n>` is a pull request if `gh issue view <n> --json url` returns a URL containing `/pull/`. Check each `#<n>` this way against the primary only, and check any issue URL's repo the same way.
   - A token that names an existing file is a file input. Check with the Read tool. Any number of file inputs is allowed.
   - The remaining text, joined with single spaces, is one ad-hoc description. It may be empty.
   - There must be at least one input of some kind.

2. Before anything else, run `gh repo view --json nameWithOwner` and reject the request, with a short one-line message and no further action, if any of these is true:
   - An input is a pull request, by URL or by `#<n>`. Pull request references are not inputs; only issues, files, and text are. To continue a pull request's branch, pass `--continue <branch>`.
   - `--continue` is given together with `--branch`.
   - `--continue <branch>` is a value that `git check-ref-format --branch <value>` rejects.
   - `--continue <branch>` names a branch that is not on the primary's selected remote. Check this on every host with `git ls-remote --heads <remote> refs/heads/<branch>`, which prints nothing when the branch does not exist. Query the exact ref: a bare branch name also matches any ref whose path ends in it. The selected remote is the one `gh repo view` resolves when it succeeds, else the one selected below. Additional `--repo` checkouts are not checked here.
   - An issue number or issue URL that gh cannot find, meaning `gh issue view <n or URL> --json url` fails. Name the input in the message.
   - An issue belongs to a different repository than the one `gh repo view` reports for the current checkout. With `--repo`, an issue URL is accepted when its owner and repo match the current checkout or any `--repo` checkout's remote (`git -C <path> remote -v`); otherwise reject it.
   - A `--repo` path that is not an existing directory or not a git checkout, checked with `git -C <path> rev-parse --git-dir`.
   - A flag is unknown, a flag value is missing or invalid, or there are no inputs.

   When `gh repo view` succeeds, the host is GitHub (GitHub Enterprise included) and the checks above apply. When it fails, run `git remote -v` and select the remote: `origin` when it exists, else the only remote. Several remotes and no `origin` is a rejection that names them. If the selected remote's URL host is `github.com`, reject with the message that `gh` is not authenticated for this remote. Otherwise the host is treated as non-GitHub: reject issue inputs only, with the message "issue inputs are not accepted on a non-GitHub host; pass a file or a description", accept file and text inputs, and continue. A GitHub Enterprise checkout without a `gh` login for that host therefore runs as non-GitHub.

   Run no other command, write no file, and do not load the skill when you reject.

3. State the parsed invocation to the user as this block, with every flag at its effective value. List each input on its own line, as `issue <#n or URL>`, `file <path>`, or `text "<description>"`.

```
mode: plan-only
inputs:
- issue <#n or URL>
- file <path>
- text "<ad-hoc description>"
flags:
  effort: auto | low | medium | high | xhigh | max
  plan-only: true
  confirm-plan: false
  no-codex: true | false
  no-publish: false
  branch: <name> | default
  continue: <branch> | none
  run-budget: <minutes> | default
  repos: <path>[, <path>...] | none
```

4. Invoke the Skill tool with skill `ccl:ccl` and that same block as the args. Then follow the skill from Step 0. Do not interpret the request, plan, or act on it yourself, and write no file: the skill records the invocation in its own first step.
