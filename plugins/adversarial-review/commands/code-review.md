---
description: Adversarial code review of recent changes. Parallel finders plus a validation pass filter false positives; reports issues without modifying code.
argument-hint: "[pr | PR number] [--post]"
---

# Code review

Review the recent changes with parallel adversarial agents and a
false-positive validation pass. You **report** findings; you do **not** modify
code. Original implementation for ZCode (not copied from Anthropic's
all-rights-reserved prompt), following the publicly known multi-agent review
pattern.

Write every user-facing report in the language the user is currently using in
this conversation (default: English). Subagent prompts are always in English.

Pass this assumption to every subagent you launch: "All tools are functional
and will work without error. Do not test tools or make exploratory calls. Only
call a tool when it is required for the task."

## Step 0 — Determine what to review

Parse the arguments: $ARGUMENTS

- If the arguments contain the keyword `pr`, the target is the current
  branch's open pull request: fetch the title and description with
  `gh pr view` and the changes with `gh pr diff` (with no number, gh targets
  the current branch's PR).
- If the arguments contain a pull request number (e.g. `123` or `#123`), the
  target is that pull request: fetch the title and description with
  `gh pr view <N>` and the changes with `gh pr diff <N>`.
- If the arguments name a branch or commit range, review that.
- Otherwise review the recent local changes:
  1. Run `git status --porcelain` and `git diff HEAD`. Uncommitted changes
     (staged, unstaged, and untracked files) take precedence when present —
     read untracked files directly.
  2. If there are no uncommitted changes, run `git diff @{upstream}...HEAD`;
     if that fails, fall back to `git diff main...HEAD`, then
     `git diff master...HEAD`, then `git diff HEAD~1`.
- `--post` only has effect together with a PR number; remember it for Step 5.

Show the user a one-line summary of the target before launching agents. If the
diff is very large (over ~1500 lines), pass each agent only the changed files
relevant to its angle plus the full file list, and say so.

## Step 1 — Collect applicable AGENTS.md files (no subagent)

Find the repository's instruction files: the root `AGENTS.md` plus any
`AGENTS.md` located in a directory containing changed files (search upward
from each changed file's directory to the repo root). Read them. If none
exist, the compliance angle in Step 2 is skipped.

## Step 2 — Launch the finders in parallel

Launch the finder subagents (Agent tool, `general-purpose`) **in a single
message** so they run concurrently. Each gets the diff, the changed file
paths, and — when the target is a PR — the PR title and description. Each
returns a list of issues, each issue with `file`, `line`, a `description`,
and the `reason` it was flagged (bug / logic / AGENTS.md rule).

- **Agent A — diff-only bug scan.** "Look ONLY at the diff; do not read
  additional context. Flag only significant bugs: code that will fail to
  compile or parse, and code that will definitely produce wrong results
  regardless of inputs. Do not flag issues you cannot validate without
  context outside the diff. Ignore nitpicks and likely false positives."
- **Agent B — introduced-code deep scan.** May explore the repository around
  the changed code. "Find problems that exist in the code introduced by these
  changes: incorrect logic, unhandled edge cases (empty, null or boundary
  values), errors swallowed or mishandled, broken contracts with callers
  (changed signatures or return semantics), and concurrency problems. Only
  look for issues that fall within the changed code."
- **Agent C — AGENTS.md compliance** (only if Step 1 found files). "Check the
  changed files against these AGENTS.md rules: <paste the applicable rules>.
  When evaluating a file, only consider rules from AGENTS.md files located at
  the file's path or a parent of it. Flag clear, unambiguous violations where
  you can quote the exact rule being broken."

High-signal bar — include in every finder prompt:

> CRITICAL: We only want HIGH-SIGNAL issues. Flag only issues you are certain
> are real. Do NOT flag: pre-existing issues outside the changed code;
> something that looks like a bug but is actually correct; pedantic nitpicks a
> senior engineer would not flag; issues a linter would catch; general
> code-quality concerns (e.g. lack of test coverage) unless an AGENTS.md rule
> explicitly requires them; issues explicitly silenced in the code (e.g.
> lint-ignore comments). If you are not certain an issue is real, do not flag
> it — false positives erode trust and waste the reviewer's time.

## Step 3 — Validate in parallel

For each issue returned by agents A and B, launch one validator subagent
(Agent tool, `general-purpose`) — all in a single message so they run
concurrently. Each validator gets the diff, the issue description (file, line,
reason), and repo read access. Its prompt:

"Your job is to validate that the stated issue is truly an issue, with high
confidence. Read the relevant code, confirm or reject the issue, and reply
with CONFIRMED or REJECTED plus one line of reasoning. Do not propose fixes
here."

For issues from agent C (AGENTS.md), the validator must additionally check
that the cited rule is actually scoped for that file and actually violated.

## Step 4 — Filter and report

Drop every issue that was not CONFIRMED. Report the survivors in the chat, in
the user's language: for each, `file:line`, severity (High / Medium / Low), a
short description, and a suggested fix. End with a one-line summary (counts by
severity). If nothing survived, say: "No issues found. Checked for bugs,
logic errors and AGENTS.md compliance." (omit AGENTS.md if it was skipped).
Do not modify any code.

## Step 5 — Post to the PR (only with `--post` AND a PR number)

Consolidate the validated findings into ONE summary comment (heading
`## Code review`, the surviving issues in the same format as Step 4, or the
no-issues sentence). Write the body to a temp file and post it:

```
gh pr comment <N> --body-file <tmpfile>
```

Then confirm to the user what was posted and link the comment.
