---
description: Plan one unit of work with the ccl loop and stop once the plan is final, changing no code. Use when the user asks to plan with ccl, or types /ccl:plan. Inputs are issue URLs or #n numbers of this repo, file paths, and a quoted description. Flags are --effort low|medium|high, --no-codex, and --branch <name>. Pull request references are rejected. To also build and publish, use /ccl:run.
argument-hint: '<#n | issue URL | file path | "description">... [--effort low|medium|high] [--no-codex] [--branch <name>]'
allowed-tools: Bash(git status:*), Bash(git rev-parse:*), Bash(git remote:*), Bash(gh repo view:*), Bash(gh issue view:*), Bash(gh pr view:*), Read, Skill
---

You are a thin forwarder for the ccl orchestrator, in plan-only mode. Do the steps below in order.

1. Parse the arguments shown between the markers below. They are what the user typed after the command. Split them into inputs and flags with these rules.

<user-text>
"$ARGUMENTS"
</user-text>

   - A flag is a token that starts with `--`. Accepted flags: `--effort`, `--no-codex`, `--branch`. `--effort` takes exactly one value, one of `low`, `medium`, `high`; without it, effort is `auto`. `--branch` takes exactly one value, a branch name; without it, branch is `default`. Reject any other flag, a missing flag value, and any `--effort` value other than `low`, `medium`, `high`. `--no-codex` takes no value. `--plan-only` is not accepted here because this command always plans only; reject it as redundant and point to `/ccl:run --plan-only`.
   - An input token that is an issue URL (`https://github.com/<owner>/<repo>/issues/<n>`) or `#<n>` is an issue. Several issues are allowed.
   - A pull request URL (`.../pull/<n>`) is a pull request. A `#<n>` is a pull request if `gh issue view <n> --json url` returns a URL containing `/pull/`. Check each `#<n>` this way, and check any issue URL's repo the same way.
   - A token that names an existing file is a file input. Check with the Read tool. Any number of file inputs is allowed.
   - The remaining text, joined with single spaces, is one ad-hoc description. It may be empty.
   - There must be at least one input of some kind.

2. Before anything else, run `gh repo view --json nameWithOwner` and reject the request, with a short one-line message and no further action, if any of these is true:
   - An input is a pull request, by URL or by `#<n>`. Pull request references are not inputs; only issues, files, and text are.
   - An issue number or issue URL that gh cannot find, meaning `gh issue view <n or URL> --json url` fails. Name the input in the message.
   - An issue belongs to a different repository than the one `gh repo view` reports for the current checkout.
   - A flag is unknown, a flag value is missing or invalid, or there are no inputs.

   Run no other command, write no file, and do not load the skill when you reject.

3. State the parsed invocation to the user as this block, with every flag at its effective value. List each input on its own line, as `issue <#n or URL>`, `file <path>`, or `text "<description>"`.

```
mode: plan-only
inputs:
- issue <#n or URL>
- file <path>
- text "<ad-hoc description>"
flags:
  effort: auto | low | medium | high
  plan-only: true
  no-codex: true | false
  branch: <name> | default
```

4. Invoke the Skill tool with skill `ccl:ccl` and that same block as the args. Then follow the skill from Step 0. Do not interpret the request, plan, or act on it yourself, and write no file: the skill records the invocation in its own first step.
