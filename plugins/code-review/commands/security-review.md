---
description: Security review of pending changes. A finder subagent hunts vulnerabilities, parallel validators kill false positives, only confidence >= 8/10 survives.
argument-hint: "[pr | PR number] [--post]"
---

# Security review

Adapted for ZCode from Anthropic's open-source security-review command
(MIT, © 2025 Anthropic):
https://github.com/anthropics/claude-code-security-review
Tool references translated for ZCode: Claude Code's `Task` tool becomes the
`Agent` tool, and dynamic context injection becomes explicit git commands.

You **report** findings; you do **not** modify code. Write every user-facing
report in the language the user is currently using in this conversation
(default: English). Subagent prompts are always in English.

## Objective

- MINIMIZE FALSE POSITIVES. Only flag issues where you're >80% confident of
  actual exploitability.
- AVOID NOISE. Skip theoretical issues, style concerns, or low-impact
  findings.
- FOCUS ON IMPACT. Prioritize vulnerabilities that could lead to unauthorized
  access, data breaches, or system compromise.
- Out of scope: DoS and resource exhaustion, secrets stored on disk when
  otherwise secured, rate limiting.

Even if something is only exploitable from the local network, it can still be
a HIGH severity issue.

## Security categories

- Input validation: SQL injection, command injection, XXE, template
  injection, NoSQL injection, path traversal.
- Authentication/authorization: auth bypass, privilege escalation, session
  flaws, JWT vulnerabilities, authorization logic bypasses.
- Cryptography and secrets: hardcoded keys, weak crypto, improper key
  storage, randomness issues, certificate validation bypass.
- Injection and code execution: RCE via deserialization, pickle injection,
  YAML deserialization, eval injection, XSS (reflected/stored/DOM).
- Data exposure: sensitive logging, PII violations, API leakage, debug
  exposure.

## Step 0 — Determine what to review

Parse the arguments: $ARGUMENTS

- If the arguments contain the keyword `pr`, the target is the current
  branch's open pull request: fetch title/description with `gh pr view`
  and changes with `gh pr diff` (with no number, gh targets the current
  branch's PR).
- If the arguments contain a pull request number (e.g. `123` or `#123`), the
  target is that pull request: fetch title/description with `gh pr view <N>`
  and changes with `gh pr diff <N>`.
- If the arguments name a branch or commit range, review that.
- Otherwise review the current branch's pending changes. Run explicitly and
  keep the outputs:
  1. `git status`
  2. `git diff --name-only @{upstream}...HEAD` (fall back to
     `origin/main...HEAD`, then `origin/master...HEAD`, then the uncommitted
     `git diff HEAD` if the branch has no upstream diff)
  3. `git log --no-decorate @{upstream}...HEAD` (same fallbacks)
  4. The full diff: `git diff --merge-base @{upstream} HEAD` (or the
     equivalent fallback; include untracked files by reading them).

Worktree safety (applies to every target, including `pr` and branch ranges):

- Run every git and `gh` command from the current working directory. Never
  `cd` to another checkout or worktree of the same repository — that would
  silently review a different branch.
- Detect whether the current directory is a linked worktree: if
  `git rev-parse --git-dir` returns something other than `.git` (or
  `git worktree list` lists more than one entry), you are in one.
- Include the working directory, `git branch --show-current` and
  `git rev-parse HEAD` in the target summary. If the tip of the diff you are
  about to review differs from the current HEAD, say so explicitly and ask
  the user to confirm before launching agents.
- Pass to the finder and every validator: "Stay inside the current working
  directory's checkout/worktree; do not cd elsewhere in the repository."

Show the user a one-line summary of the target before launching agents.

## Step 1 — Launch the finder subagent

Launch ONE finder subagent (Agent tool, `general-purpose`); it may explore
the repository for context. Include in its prompt ALL of the following: the
Objective, the Security categories, this methodology, and the actual git
status + diff outputs.

Analysis methodology:

1. **Repository context research.** Use file exploration tools to understand
   the codebase: what security frameworks are in use, what secure patterns
   are already established, and the project's implicit threat model.
2. **Comparative analysis.** Compare the new code against the repository's
   existing security patterns. Flag deviations and newly introduced attack
   surfaces.
3. **Vulnerability assessment.** Examine each modified file. Trace data flow
   from user inputs to sensitive operations. Look for unsafe
   privilege-boundary crossings.

Finder output format — one block per finding:

```
# Vuln N: <category>: `<file>:<line>`
* Severity: High|Medium|Low
* Description: ...
* Exploit Scenario: ...
* Recommendation: ...
```

Severity: HIGH = remote code execution, data breach, or auth bypass;
MEDIUM = significant impact under specific conditions; LOW =
defense-in-depth. Confidence scale: 0.9–1.0 certain, 0.8–0.9 clear
vulnerability pattern, 0.7–0.8 suspicious. Below 0.7: don't report (too
speculative). Focus on HIGH and MEDIUM findings only.

## Step 2 — Launch parallel false-positive validators

For EACH vulnerability the finder reported, launch one validator subagent
(Agent tool, `general-purpose`) — all in a single message so they run
concurrently. Each validator gets the finding plus repo read access, and
this FALSE POSITIVE FILTERING block:

> You do not need to run commands to reproduce the vulnerability; just read
> the code to determine if it is a real vulnerability. Do not use the bash
> tool and do not write to any files.
>
> HARD EXCLUSIONS — reject the finding if it is any of these: DoS or resource
> exhaustion; secrets on disk when otherwise secured; rate limiting; memory
> or CPU exhaustion; input validation on non-security-critical fields;
> GitHub Actions input sanitization unless clearly triggerable by untrusted
> input; a mere lack of hardening (only flag concrete vulnerabilities);
> theoretical race conditions; outdated third-party libraries; memory-safety
> issues in memory-safe languages; test-only files; log spoofing (outputting
> unsanitized user input to logs is not a vulnerability); SSRF that only
> controls the path (SSRF matters only if it can control host or protocol);
> user content in AI system prompts; regex injection or ReDoS; documentation
> files; lack of audit logs.
>
> PRECEDENTS — logging high-value secrets in plaintext IS a vulnerability,
> logging URLs is assumed safe; UUIDs can be assumed unguessable; environment
> variables and CLI flags are trusted values, so any attack that relies on
> controlling an environment variable is invalid; low-impact web issues
> (tabnabbing, XS-Leaks, prototype pollution, open redirects) only count at
> extremely high confidence; React/Angular XSS only via dangerouslySetInnerHTML
> or bypassSecurityTrustHtml; client-side permission checks alone are not a
> vulnerability; include MEDIUM findings only if they are obvious and concrete
> issues; notebook vulnerabilities need concrete attack paths; logging
> non-PII is not a vulnerability; shell-script command injection is generally
> not exploitable without untrusted input.
>
> SIGNAL QUALITY — ask: 1. Is there a concrete, exploitable vulnerability
> with a clear attack path? 2. Does this represent a real security risk vs
> theoretical best practice? 3. Are there specific code locations and
> reproduction steps? 4. Would this finding be actionable for a security
> team?

Each validator replies with a confidence score from 1 to 10 plus one line of
reasoning.

## Step 3 — Filter and report

Keep ONLY the vulnerabilities whose validator scored confidence >= 8. Report
them in the chat, in the user's language, using the finder's output format
(file:line, severity, category, description, exploit scenario,
recommendation). If none survive, say: "No security issues found (reviewed
with false-positive filtering)." Do not modify any code.

## Step 4 — Post to the PR (only with `--post` AND a PR number)

Consolidate the surviving findings into ONE summary comment (heading
`## Security review`). Write the body to a temp file and post it:

```
gh pr comment <N> --body-file <tmpfile>
```

Then confirm to the user what was posted and link the comment.
