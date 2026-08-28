---
description: Full adversarial pipeline on recent changes. Simplify first (applies fixes), then code review and security review (report findings), then a consolidated summary.
argument-hint: "[pr | PR number] [--post]"
---

# Deep review

Run the full adversarial pipeline over one target, strictly in this order:
**simplify → code review → security review**, ending with a consolidated
summary. The order is deliberate: simplify applies its fixes first, so the two
reviews analyze the final cleaned code instead of wasting findings on
complexity that was about to disappear.

This command is self-contained — execute each stage's workflow as specified
below. Stage 1 modifies code; stages 2 and 3 only report. Write every
user-facing report in the language the user is currently using in this
conversation (default: English). Subagent prompts are always in English. All
subagents are launched with the Agent tool (`general-purpose`); whenever a
stage says "in parallel", launch them in a single message.

## Step 0 — Determine the target (once, for all stages)

Parse the arguments: $ARGUMENTS

- If the arguments contain the keyword `pr`, the target is the current
  branch's open pull request: fetch title/description with `gh pr view`
  and changes with `gh pr diff` (with no number, gh targets the current
  branch's PR). Remember `--post` for the final step.
- If the arguments contain a pull request number (e.g. `123` or `#123`), the
  target is that pull request: fetch title/description with `gh pr view <N>`
  and changes with `gh pr diff <N>`. Remember `--post` for the final step.
- If the arguments name a branch or commit range, review that.
- Otherwise review the recent local changes:
  1. `git status --porcelain` + `git diff HEAD`; uncommitted changes
     (including untracked files, read directly) take precedence when present.
  2. Otherwise `git diff @{upstream}...HEAD`, falling back to
     `git diff main...HEAD`, then `master...HEAD`, then `HEAD~1`.

Show the user a one-line summary of the target, then announce each stage as
you start it (brief progress notes are enough; the full detail goes in the
final summary).

## Stage 1 — SIMPLIFY (applies changes)

1. Launch **4 parallel review agents** on the diff, one per angle:
   - **Reuse** — changed code that re-implements existing repo helpers or
     copy-pasted blocks that could share one implementation (name the
     duplicated code).
   - **Simplification** — unnecessary nesting, single-caller abstractions,
     dead code, always-true conditions, misleading names.
   - **Efficiency** — concrete wasted work (recomputation in loops, repeated
     I/O that could batch, O(n²) where O(n) is trivial). No
     micro-optimizations.
   - **Altitude** — wrong level of abstraction, responsibilities in the wrong
     module or layer.
   Each agent: only changed lines, no bugs/security/style (other stages own
   those), and returns `file`, `line`, one-line `summary`, concrete `cost` —
   or exactly "No findings."
2. Dedup findings pointing at the same line or mechanism; apply each fix
   directly. Skip (with a one-line reason) fixes that would change intended
   behavior, require changes well outside the diff, or look like false
   positives.
3. If the repo has a fast test/lint/typecheck/build command, run it; revert
   any simplification that broke something and record it as skipped.
4. **Regenerate the diff** — stages 2 and 3 must review the
   post-simplification state.

## Stage 2 — CODE REVIEW (report only)

1. Collect applicable `AGENTS.md` files (repo root plus any in directories of
   changed files); skip the compliance finder if none exist.
2. Launch **3 parallel finders**, each getting the regenerated diff, changed
   file paths, and the PR title/description when available:
   - **A — diff-only bug scan.** Only the diff, no extra context; only
     significant bugs (won't compile/parse, or definitely wrong results
     regardless of inputs); nothing that needs out-of-diff context to validate.
   - **B — introduced-code deep scan.** May explore the repo. Incorrect
     logic, unhandled edge cases, swallowed errors, broken caller contracts,
     concurrency — only within the changed code.
   - **C — AGENTS.md compliance.** Only clear violations where the exact rule
     can be quoted, and only rules from AGENTS.md files at or above the
     file's path.
   High-signal bar for all finders: only flag issues you are CERTAIN are
   real. No style preferences, no pre-existing issues, no linter territory,
   no "lack of tests" unless a rule requires it, no silenced-by-comment
   issues. False positives erode trust.
3. For each issue from A and B launch **parallel validators** (C's validators
   re-check rule scope and violation): CONFIRMED or REJECTED with one line of
   reasoning. Drop everything not CONFIRMED. Keep the survivors for the final
   summary. Do not modify code.

## Stage 3 — SECURITY REVIEW (report only)

1. Launch **one finder** (may explore the repo) with the regenerated diff and
   these categories: input validation (SQLi, command injection, XXE, template
   injection, NoSQL injection, path traversal), auth/authz bypasses, crypto
   and secrets, RCE via deserialization/eval, XSS, data exposure. Objective:
   minimize false positives (>80% confidence), skip theoretical or low-impact
   findings, prioritize unauthorized access / data breach / system
   compromise. Methodology: research repo security patterns, compare new code
   against them, trace data flow from user inputs to sensitive operations.
   Findings as: file:line, severity (HIGH/MEDIUM/LOW), category, description,
   exploit scenario, recommendation. Local-network-exploitable still counts
   as HIGH.
2. For EACH finding launch **parallel validators** with the false-positive
   rules: read-only (no bash, no writing); reject DoS/resource exhaustion,
   secrets-on-disk-when-secured, rate limiting, theoretical races, outdated
   deps, test-only files, log spoofing, path-only SSRF, ReDoS, docs, audit
   logging; env vars and CLI flags are trusted; UUIDs unguessable; log URLs
   safe, plaintext high-value secrets not; MEDIUM only when obvious and
   concrete. Score confidence 1–10.
3. Keep ONLY findings with confidence >= 8. Do not modify code.

## Stage 4 — Consolidated summary (user's language)

One final report:

1. **Simplify**: what was fixed (grouped by angle), what was skipped and why,
   verification result.
2. **Code review findings** (validated only): `file:line`, severity,
   description, suggested fix — pending the user's decision.
3. **Security findings** (confidence >= 8 only): `file:line`, severity,
   category, exploit scenario, recommendation — pending the user's decision.
4. State explicitly that review findings were NOT applied — the user decides
   what to act on. Offer to apply any of them on request.

If `--post` was passed AND the target is a PR: consolidate everything into
ONE comment (heading `## Deep review`), write it to a temp file, post with
`gh pr comment <N> --body-file <tmpfile>`, and link the comment.
