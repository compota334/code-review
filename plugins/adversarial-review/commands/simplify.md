---
description: Simplify recently changed code. Four adversarial agents (reuse, simplification, efficiency, altitude) review it, then the safe fixes get applied.
argument-hint: "[pr, PR number, branch, or file path]"
---

# Simplify

You are improving the **quality** of the recently changed code, not hunting for
bugs. Review the changes for reuse, simplification, efficiency and altitude
issues, then fix what you find. Do not look for correctness bugs or security
issues — that is what `/code-review` and `/security-review` are for.

Original implementation for ZCode, following the publicly known multi-agent
pattern of Claude Code's bundled `/simplify` (whose prompt Anthropic does not
publish).

Write every user-facing report in the language the user is currently using in
this conversation (default: English). Subagent prompts are always in English.

## Step 0 — Determine what to review

Parse the arguments: $ARGUMENTS

- If the arguments contain the keyword `pr`, the target is the current
  branch's open pull request: fetch it with `gh pr diff` (and `gh pr view`
  for context — with no number, gh targets the current branch's PR).
- If the arguments contain a pull request number (e.g. `123` or `#123`), the
  target is that pull request: fetch it with `gh pr diff <N>` (and `gh pr view <N>`
  if context helps).
- If the arguments name a branch, commit range or file path, review that.
- Otherwise review the recent local changes:
  1. Run `git status --porcelain` and `git diff HEAD`. Uncommitted changes
     (staged, unstaged, and untracked files) take precedence when present —
     read untracked files directly.
  2. If there are no uncommitted changes, run `git diff @{upstream}...HEAD`;
     if that fails, fall back to `git diff main...HEAD`, then
     `git diff master...HEAD`, then `git diff HEAD~1`.

Show the user a one-line summary of the target (files changed, rough size)
before launching agents. If the diff is very large (over ~1500 lines), pass
each agent only the changed files relevant to its angle plus the full file
list, and say so.

## Step 1 — Launch 4 parallel review agents

Launch 4 independent review subagents (Agent tool, `general-purpose`) **in a
single message** so they run concurrently. Each subagent gets: the diff (or its
relevant slice), the list of changed file paths, repo read access for context,
and exactly ONE of the four angles below.

Common instructions for every subagent:

- Only report findings inside the changed lines of the diff.
- Do NOT report correctness bugs, security issues, or style/formatting
  preferences — those belong to other commands.
- Return findings as a list; each finding must have `file` (path), `line`,
  a one-line `summary`, and the concrete `cost` — what is duplicated, wasted,
  or harder to maintain because of it.
- If nothing is found, reply exactly "No findings."

The four angles:

- **Reuse** — Find changed code that re-implements something the repository
  already provides: existing helpers, utilities, wrappers or constants that do
  the same job, and copy-pasted blocks inside the diff that could share one
  implementation. For each finding, name the existing code being duplicated
  (file and symbol). Explore the repo to know what exists.
- **Simplification** — Find unnecessary complexity in the changed code: deep
  nesting that early returns would flatten, abstractions with only one caller
  or one use case, dead code (unreachable branches, unused variables, imports
  or parameters), conditions that are always true or redundant, and names that
  misrepresent what the code does. Only flag complexity that can be removed
  without changing behavior.
- **Efficiency** — Find clearly wasted work in the changed code: recomputing
  the same value inside a loop, repeated I/O or network calls that could be
  batched or hoisted, O(n²) patterns where an O(n) alternative is trivial,
  loading a whole collection to use one element. Do NOT flag
  micro-optimizations or hypothetical performance — only concrete waste.
- **Altitude** — Find code operating at the wrong level of abstraction:
  high-level policy buried inside low-level helpers, low-level details leaking
  into orchestration code, responsibilities placed in the wrong module or
  layer (judging by the project's structure), and data threaded through
  several layers when the decision could be made where the information lives.

## Step 2 — Deduplicate and apply

Wait for all 4 agents. Merge findings that point at the same line or the same
mechanism (keep the better-articulated one). Then fix each remaining finding
directly by editing the files.

Skip — and do not argue with — any finding whose fix would:

- change the intended behavior of the code,
- require changes well outside the reviewed diff, or
- be, in your judgment, a false positive.

Note every skip with a one-line reason for the final summary.

## Step 3 — Verify

If the project has a fast verification command (unit tests, lint, typecheck or
build — detect it from the repo config; skip anything that would take
minutes), run it now. If a simplification broke something, revert that
specific simplification and record it as skipped.

## Step 4 — Final summary

Report briefly, in the user's language: what was fixed (grouped by angle),
what was skipped and why, and the verification result. End by noting that
correctness and security were deliberately out of scope.
