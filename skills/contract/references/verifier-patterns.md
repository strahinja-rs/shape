# Verifier Patterns

Use this reference when a task has no obvious test command or when the work spans code, UI, research, documents, data, or operations.

## Universal pattern

Turn each slice into:

```markdown
Claim:
Observable state:
Verifier:
Pass threshold:
Evidence:
Failure diagnosis:
```

A useful verifier produces an observable result outside the model's private reasoning. Prefer machine-checkable outputs. When that is not possible, use a written rubric plus an independent review, and label it as judgment-based.

## Coding tasks

Good verifiers:

- Focused unit or integration tests for the changed behavior.
- Existing test suite around the touched module.
- Typecheck, lint, formatting, build, migration validation.
- CLI/API smoke tests with real inputs and captured outputs.
- Diff review against the requested behavior.

Loop sequence:

1. Reproduce or inspect the current behavior.
2. Add or identify the narrowest failing check.
3. Change code.
4. Run the narrow check.
5. Run adjacent checks.
6. Review diff for regressions and missing tests.

## Frontend and visual tasks

Good verifiers:

- Browser smoke test of the actual local route via Playwright (`mcp__plugin_playwright_playwright__*`), Chrome MCP (`mcp__Claude_in_Chrome__*`), or Claude Preview (`mcp__Claude_Preview__*`).
- Screenshot at desktop and mobile sizes.
- Console error check (`browser_console_messages` / `preview_console_logs`).
- Network request inspection for failed calls.
- Interaction checks for the main controls (click, type, submit).
- Layout checks for overlap, clipping, blank canvases, broken assets, and responsive behavior.

Evidence should include route, viewport sizes, screenshots if useful, console status, and any interactions performed.

For desktop apps where no browser is involved, use `mcp__computer-use__*` for screenshots and interaction; treat the captured screenshots as evidence.

## Research tasks

Good verifiers:

- Source list with URLs and access dates when recency matters.
- Primary-source coverage for technical, legal, medical, financial, or product facts.
- Contradiction check across independent sources.
- Explicit separation of sourced facts, inference, and recommendation.
- Coverage matrix against the user's questions.
- Quote limit and citation check.

Loop sequence:

1. Convert the question into subquestions.
2. Define what a good answer must cover.
3. Search primary or authoritative sources first.
4. Record sources and dates.
5. Draft synthesis.
6. Review for unsupported claims, missing angles, and stale facts.

When delegating research to a subagent (`subagent_type: "Explore"` or `"general-purpose"`), pass the coverage matrix and source-list requirement as part of the prompt so the subagent produces evidence the parent can verify.

## Documents and content

Good verifiers:

- Required sections present.
- Audience, tone, and length match the request.
- Links, citations, references, and cross-references resolve.
- Tables, headings, and numbering are structurally valid.
- Rendered output inspection for PDFs, slides, DOCX, or HTML.
- Rubric review for clarity, completeness, and actionability.

For subjective writing, verify the objective shell first: structure, completeness, constraints, claims, citations, and absence of contradictions. Leave taste, preference, and final editorial judgment as human-only unless the user supplied a rubric.

## Data and spreadsheet tasks

Good verifiers:

- Schema and column checks.
- Row counts before and after transformation.
- Formula recalculation and sampled cell inspection.
- Pivot/chart source validation.
- Null, duplicate, range, and type checks.
- Export/open/render check for workbook artifacts.

Evidence should include file path, sheet/range, row counts, formulas touched, and sampled outputs.

For Mongo / Postgres / live data work, prefer a small read-only query that asserts row counts, schema shape, or sampled values over "I edited the document" claims.

## Operations and deployment tasks

Good verifiers:

- Dry-run mode or plan output.
- Environment and target confirmation (which project, which cluster, which branch).
- Config validation.
- Health check or smoke test after change.
- Rollback path identified before destructive action.
- Logs from the affected service.

Human gates are required for irreversible production data changes, payment-affecting actions, broad permission grants, credential entry, or external communications unless the user explicitly preauthorized them.

## Anti-gaming checks

Before declaring pass, ask:

- Did any verifier get weakened, skipped, or replaced by a less relevant check?
- Was success achieved by hardcoding the visible example instead of implementing the general behavior?
- Did the loop ignore a failing adjacent check?
- Did the final claim exceed the evidence?
- Did a reviewer approve without seeing the task, diff or artifact, and verifier results?

If any answer is yes, mark the task partially verified or continue the loop.

## Flaky or weak verifiers

If a verifier is flaky, nondeterministic, or weaker than the claim:

- Rerun once to confirm the behavior.
- Capture the exact command, seed, environment, timing, and failure mode when available.
- Narrow the verifier or add instrumentation before broad retries.
- Do not turn a flaky pass into a clean pass; report it as partially verified.
- Prefer a stronger adjacent verifier before asking a reviewer to decide.

## No-progress rule

Stop or change strategy when two consecutive attempts on the same slice produce the same failure without new evidence. A good next action is one of:

- Narrow the slice.
- Add instrumentation.
- Read the relevant source or docs.
- Ask the smallest blocking question.
- Escalate to a reviewer with the failure evidence.
