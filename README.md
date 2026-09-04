# code-review

code-review is a review suite for AI coding agents: the
/simplify, /code-review and /security-review you know from Claude Code,
rebuilt for ZCode and improved. Clean-context subagents that didn't write
the change review your latest diff, parallel validators kill false
positives before they reach you, and /deep-review runs the whole pipeline
in one command. The commands you know, judging code they didn't write. For
ZCode.

## Where it comes from

Claude Code ships three review commands: `/simplify` applies quality fixes,
`/code-review` hunts bugs, `/security-review` hunts vulnerabilities
(prompt open-sourced by Anthropic under MIT). This plugin brings that
toolkit to ZCode and pushes it further:

- **`/deep-review` doesn't exist in Claude Code**: it chains the three
  commands in the right order (simplify first, applying its fixes, so both
  reviews analyze the final cleaned code) and closes with one consolidated
  summary.
- **Smarter targets**: `pr` for the current branch's PR, a PR number, a
  branch, a commit range or a file path, and a default scope of
  "uncommitted changes first, unpushed commits second" that matches what
  you just worked on.
- **Reports in the language of the conversation**, not hardcoded English.
- **`--post`** publishes one consolidated summary comment on the PR.

What did not carry over verbatim: Anthropic's `/code-review` prompt is
all-rights-reserved (this is an original implementation of the same
multi-agent architecture), their `/simplify` prompt was never published
(original implementation, same known pattern), and `/security-review` is
adapted from the MIT original with attribution.

## The core idea: judgment without the author's bias

The agent that wrote the code reviews it the way you proofread your own
email: seeing what was meant, not what is there. So every review here is
delegated to subagents launched with **clean context**: they never saw the
conversation, and the diff has to prove everything.

- **The target is always a diff, never the codebase.** Findings are only
  ever reported on changed lines; the rest of the repo is context the
  agents may read, never territory they may flag.
- **Finders work one narrow angle each, in parallel.** A narrow angle with
  a high bar beats a general glance.
- **Nothing reaches you unvalidated.** Every finding gets its own fresh
  validator subagent that confirms or rejects it; security findings must
  additionally score confidence >= 8/10 against a hard-exclusion list.
- **Simplify applies, reviews report.** You keep the decision: the final
  summary lists what changed and what was found, and offers to apply the
  review findings on request.

## The two problems it solves

1. **The author bias.** The agent that wrote the code reviews it with the
   confidence of the one who wrote it. Delegating to clean-context subagents
   removes the shared assumptions, not just the tone.
2. **The noise problem.** LLM reviewers over-flag, and a wall of theoretical
   findings trains you to ignore all of them. The validation pass and
   explicit thresholds mean everything that lands in your chat was confirmed
   by an agent whose only job was to doubt it.

## The four commands

| Command | What it does | Touches code? |
|---|---|---|
| `/simplify` | 4 parallel agents review the diff (reuse, simplification, efficiency, altitude); the safe fixes get applied and verified | Yes |
| `/code-review` | Parallel finders (diff-only bugs, introduced-code deep scan, AGENTS.md compliance) plus one validator per finding | No, reports only |
| `/security-review` | One vulnerability finder plus one false-positive validator per finding; only confidence >= 8/10 survives | No, reports only |
| `/deep-review` | The whole pipeline in order: simplify → code review → security review → consolidated summary | Stage 1 only |

### What each command actually does

- **`/simplify`** launches four agents in parallel, each with one angle:
  reuse (duplicated logic the repo already provides), simplification
  (nesting, dead code, single-caller abstractions), efficiency (concrete
  wasted work, never micro-optimizations), altitude (wrong level of
  abstraction). Each returns findings with a concrete cost. The orchestrator
  dedups, applies every fix that cannot change behavior, skips the rest
  with a stated reason, and verifies with the project's fast checks.
- **`/code-review`** runs three finders: a diff-only bug scan (only bugs
  provable from the diff), a deep scan with repo context (logic, edge
  cases, swallowed errors, broken caller contracts, concurrency), and an
  AGENTS.md compliance check. Every finding then gets its own validator;
  only confirmed issues reach you, each with file:line, severity and a
  suggested fix.
- **`/security-review`** sends one finder across the classic categories:
  injection (SQL/command/NoSQL/XXE/template), auth and authorization,
  crypto and secrets, RCE via deserialization, XSS, data exposure, with a
  methodology of tracing data flow from user inputs to sensitive
  operations. Each finding faces a read-only validator armed with the
  false-positive exclusions; confidence below 8/10 is dropped, and only
  HIGH and MEDIUM severities are reported.
- **`/deep-review`** runs simplify first (it modifies code), regenerates
  the diff, then runs both reviews on the simplified result and merges
  everything into one summary: what changed, what was found, what is
  waiting for your decision.

## What gets reviewed (the scope)

All four commands accept a target:

- **No arguments**: your uncommitted changes (untracked files included),
  falling back to the diff against `@{upstream}`, then `main`, then the last
  commit. This is "the latest thing I did", by construction.
- **`pr`**: the current branch's open pull request; `gh pr diff` with no
  number resolves it: `/deep-review pr`.
- **A PR number**: `/code-review 123` reviews PR #123.
- **A branch, commit range or file path**: `/simplify main..feature` or
  `/simplify src/app.py`.
- **`--post`**: with a PR target, additionally publishes one consolidated
  summary comment on the PR via `gh pr comment`.

## Worktree safety

If your work lives in a git worktree (common with parallel agent "lanes"),
a review command that resolves the repository "helpfully" can end up in the
main checkout and silently review the wrong branch. Every command here is
built to prevent that:

- All git and `gh` commands run from the directory the session is in; the
  commands explicitly forbid `cd`-ing to another checkout or worktree of
  the same repository.
- Before launching any agent, the command detects whether it is inside a
  linked worktree (`git rev-parse --git-dir` / `git worktree list`) and
  shows you the working directory, branch and HEAD it is about to review.
- If the diff's tip doesn't match the current worktree's HEAD, the command
  stops and asks you to confirm instead of reviewing ahead.
- Every subagent (finders, validators, simplify agents) receives the same
  "stay inside this worktree" instruction.

## Install

### Option 1: From GitHub

**Settings → Plugin Management → Discover → `+`** → add
`https://github.com/compota334/code-review` as a marketplace →
install **code-review** → restart ZCode (plugins resolve at startup).

### Option 2: From a local directory

Same **`+`** button, pointing at the repository root (the folder containing
`marketplace.json`) instead of a URL. Install and restart.

## Credits and licenses

- `/security-review` is adapted from Anthropic's open-source
  [claude-code-security-review](https://github.com/anthropics/claude-code-security-review)
  (MIT, © 2025 Anthropic PBC): same finder → validator → threshold
  architecture and hard exclusions, translated for ZCode (`Task` → `Agent`,
  injected git context → explicit commands).
- `/code-review` and `/simplify` are original implementations of the
  publicly known multi-agent review pattern.

MIT; see [LICENSE](LICENSE).
