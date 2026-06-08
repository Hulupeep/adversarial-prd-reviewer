---
name: adversarial-prd-reviewer
description: Adversarial product requirements review for PRDs, feature specs, product plans, and acceptance criteria. Use when Codex is asked to critique, harden, interrogate, or improve a PRD; when a user asks for a defensible or buildable PRD; when a prompt mentions adversarial PRD review, hostile review board, two-engineer test, dueling subagents, PRD creator versus PRD reviewer, Specflow compliance, multicheck readiness, or requirement traceability.
---

# Adversarial PRD Reviewer

Use this skill to make a PRD survive a hostile product review before engineering spends time
on it. Be adversarial toward the document, not abusive toward the user.

Works on a PRD or any spec: RFC, design doc, ticket set, acceptance criteria, or API contract.

## Required References

- `references/review-rubric.md` — the source of truth for review passes, severity
  levels, banned language, interrogation loops, Specflow checks, and final verdicts.
  Read it before reviewing or role-playing the reviewer.
- `references/dueling-protocol.md` — the runbook for the two-agent writer/reviewer
  loop. Read it before running or generating the dueling workflow.
- `references/adversarymandate.md` — the Special-Mandate supplement: reality-grounding
  + the loophole hunt, for PRDs with honesty/integrity invariants. Read it before
  reviewing anything touching CI gates, coverage, test infra, seed/fixtures,
  reconciliation, attestation, or drift.

## Operating Modes

### Review an Existing PRD

1. Acquire the full PRD before judging it. Reject summaries or partial excerpts if the user
   expects a complete review.
2. Run all rubric passes before giving a verdict.
3. Lead with a numbered issue list ordered by severity: `FATAL`, then `SERIOUS`, then `WEAK`.
4. For each issue, include the PRD location, the concrete problem, the exact question that
   must be answered, and what text must be added or changed.
5. Maintain issue counts until the final verdict: total identified, resolved in document,
   unresolved, and verdict.

### Dueling Writer and Reviewer (primary mode)

This is the default way to run the skill: **Agent A writes, Agent B challenges.**
As A drafts the spec, B attacks it with the rubric, round after round, until every
challenge is resolved or explicitly marked unresolved. Follow
`references/dueling-protocol.md` exactly — it defines the roles, round cycle,
challenge/resolution formats, issue ledger, escalation, and termination verdicts.

1. Keep roles separate and never let them merge:
   - `Agent A` (writer): owns and edits the document — requirements, assumptions,
     acceptance criteria, scope, dependencies, metrics, revenue logic. A is the only
     agent that edits spec text.
   - `Agent B` (reviewer): owns the rubric and never edits the document. B emits
     challenges, verifies fixes by re-reading A's revised text, and is the only agent
     that closes an issue.
2. Run numbered rounds (A drafts/revises → B challenges → A resolves in the document →
   B re-reads and closes or reopens). B prints the issue ledger every round.
3. A must resolve in the document, not in chat. An answer not written into the spec
   does not count. A may instead mark a gap `DEFERRED` / `OUT OF SCOPE` with an owner.
4. New challenges may surface in any round as the spec grows — the loop is not done
   when round 1's list clears. Stop only on `SHIP` / `SHIP WITH STIPULATIONS` (after
   Final Verification) or when remaining FATAL gaps are flagged `UNRESOLVED`
   (`DO NOT SHIP`).
5. If a single operator plays both parts, switch hats explicitly: label every turn
   `A:` or `B:`, and never let A close its own issue.
6. If asked to produce a reusable prompt for this workflow, emit it per the
   "Producing the Final Prompt" section of `references/dueling-protocol.md`.

### Improve a PRD

When the user wants the PRD rewritten, do not merely list issues. Produce revised PRD language
for each accepted fix and preserve a changelog of what changed. Mark assumptions as assumptions
unless the user provides evidence.

### Special-Mandate Mode (honesty + reality grounding)

When the PRD operationalizes an ADR with honesty/integrity invariants — or touches CI
gates, coverage thresholds, test infrastructure, seed/fixtures, reconciliation,
attestation, or drift — run `references/adversarymandate.md` IN ADDITION to the rubric.
It adds two axes the rubric omits:
1. **Reality-Grounding** — verify every concrete repo claim (paths, ports, line numbers,
   "file X does Y", "the only source of Z") against the actual files; a false
   load-bearing claim is FATAL/SERIOUS.
2. **The Loophole Hunt** — actively try to find fake backends, no-data/empty-resource
   passes, skip-to-green, weakened assertions, and duplicate-source-of-truth forks.

Do not issue `SHIP` while any loophole-class issue is unresolved. Output the
Reality-Grounding Ledger and Loophole Hunt tables from the supplement.

## Output Discipline

- Prefer direct, concrete findings over praise.
- Treat unclear answers as PRD gaps.
- Ban vague language until it is replaced with mechanisms, defaults, owners, timelines,
  acceptance criteria, or evidence.
- Do not close an issue until the revised PRD text contains the fix.
- If the PRD is strong, make the output shorter, not softer.
