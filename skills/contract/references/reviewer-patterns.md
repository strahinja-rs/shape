# Reviewer Patterns

Use reviewers after there is a task, contract, artifact or diff, and verifier evidence. A reviewer should produce findings, not edit files.

## Claude self-review

Use this for most code and artifact changes before final delivery. `/fresh-eyes` is the dedicated skill; the manual pattern below is the underlying shape.

Review prompt shape:

```markdown
Review the work against this Loop Contract.

Task:
Contract:
Changed files or artifact:
Verifier evidence:

Find only concrete issues that could cause incorrect behavior, regressions, missing coverage, security or privacy risk, or unsupported claims.
Return findings first, ordered by severity, with file and line references when applicable.
If there are no issues, say so and name remaining test gaps.
```

## Subagent review (Agent tool)

Use a subagent when a fresh bounded read can run in parallel or when the task benefits from independent review without polluting the main context.

Pick the right `subagent_type`:

- **`Explore`** — fast read-only search agent. Use for "find / locate / which files reference X" review questions. Not for design or open-ended analysis.
- **`general-purpose`** — multi-step independent review. Use for cross-file consistency checks, design-doc audits, code review.
- **`Plan`** — software architect agent. Use to second-opinion an implementation plan before committing to it.
- A specialized agent (e.g. `code-reviewer` if installed) — use when its description matches the review scope.

Give the subagent:

- The user task.
- The Loop Contract.
- The changed files, diff, artifact path, or research draft.
- The verifier evidence.
- A narrow review question.

Do NOT give the subagent your intended answer or suspected bug unless the review specifically requires confirmation. The point is an independent read.

Cap parallel subagents at ~8 per the orchestrator anti-pattern. Beyond that, split into multiple sessions.

## External CLI review (Codex / GPT-5.4)

Use a different model as a read-only external reviewer when the task is high-risk, broad, architecture-heavy, security-sensitive, ambiguous, expensive to regress, or ready for final delivery. Prefer deterministic verifiers first, then external review for adversarial perspective.

### Codex CLI

Three slash-skills wrap the local Codex runtime:

- **`/codex:rescue`** — when stuck, want a second implementation pass, or need a deeper root-cause investigation. Hands a substantial task to Codex through the shared runtime.
- **`/codex:review`** — standard code review pass by Codex.
- **`/codex:adversarial-review`** — hostile review for finding security, regression, and edge-case risk.

These are the right default for code-shaped review. Codex runs locally and operates on the same filesystem, so it can read the actual diff without you pasting it.

### GPT-5.4 Extended Thinking

`/gpt54` packages the current context into an optimized prompt for ChatGPT 5.4 Pro Extended Thinking. Use for:

- Deep reasoning tasks (architecture decisions, complex correctness arguments, multi-constraint optimization).
- Cases where you want a non-Anthropic model's perspective.
- High-cost-of-error decisions where a fresh model + extended thinking is worth the round-trip.

Output is a prompt to paste into ChatGPT; the user runs it manually and brings findings back.

### Raw `claude` CLI (cross-instance review)

Available as a fallback if you want a fresh Claude instance with no session state. Read-only pattern:

```bash
claude -p "$PROMPT" \
  --model claude-opus-4-7 \
  --output-format json \
  --permission-mode dontAsk \
  --allowed-tools "Read,Glob,Grep" \
  --disallowed-tools "Edit,Write,WebFetch,Bash" \
  --max-turns 6 \
  --max-budget-usd 2.00 \
  --no-session-persistence \
  --add-dir "$REVIEW_ROOT"
```

Use `--add-dir` only for the smallest directory the reviewer needs to inspect. If the local Claude CLI does not support a flag in the pattern, omit it and record that the external review ran with a reduced guardrail. If the command exits nonzero, record the review as not run; do not silently replace it with a passed review.

Do NOT use `--dangerously-skip-permissions` or any mode that allows edits for review.

## Security: what NOT to send

Before pasting any excerpt, diff, or path into an external review prompt, scan for:

- `secret`, `token`, `key`, `password`, `.env`, credentials, high-entropy strings.
- Customer data, private messages, PII.
- Unrelated files outside the review scope.

Prefer sending paths (inside `--add-dir` for `claude` CLI, or by reference for Codex) over pasting raw sensitive content. Redact anything questionable.

## External review prompt shape

```markdown
You are a read-only reviewer. Do not edit files or run write commands.

Task:
<user task>

Loop Contract:
<contract>

Evidence:
<commands run, results, screenshots or artifact paths>

Review surface:
<diff summary, file paths, artifact path, or pasted excerpt>

Find concrete issues only. Focus on correctness, regressions, missing tests, security/privacy risk, unsupported claims, and places where the evidence does not support completion.

Return JSON with:
{
  "verdict": "pass | concerns | fail",
  "findings": [
    {
      "severity": "critical | high | medium | low",
      "location": "file:line or artifact section",
      "issue": "...",
      "evidence": "...",
      "suggested_fix": "..."
    }
  ],
  "missing_verification": ["..."],
  "residual_risk": ["..."]
}
```

For Codex via `/codex:review` or `/codex:adversarial-review`, the skill handles the prompt shaping. Pass the Loop Contract and verifier evidence as part of the user message.

## Acting on findings

Treat external findings as **review input**, not as truth:

1. Confirm any concrete issue against the local files or artifact before changing code.
2. Rerun relevant verifiers after changes.
3. If a finding cannot be reproduced at the cited location or from the cited evidence, log it as unverified and do not act on it.
4. Reviewer findings without confirmable evidence are not authoritative.

## When NOT to escalate to a reviewer

- The deterministic verifier failed and the fix is clear. Fix it; don't ask a reviewer.
- Two consecutive attempts on the same slice failed the same way without new evidence. Per No-Progress Rule: narrow, add instrumentation, or ask a blocking question, not "run it past Codex".
- The slice has not been verified yet. Reviewers find missed risks in evidence; they don't generate evidence.
