# The Full Pipeline — from rough idea to *defensible* epics

This shows how the Adversarial PRD Reviewer fits into an end-to-end practice that turns a rough
idea (or a half-built repo) into **GitHub epics and issues that are defensible, not perfect** —
they survived a hostile review, their gaps are named with owners, and nothing pretends to be
done that isn't.

It pairs with **[Specflow](https://github.com/Hulupeep/Specflow)** — install that as the
companion that turns the hardened spec into tickets and keeps them *enforceable forever*
(contracts + journey tests + CI gates).

---

## The loop

```
   rough idea  /  existing repo  /  "we already built half of this"
            │
            ▼
   1. DRAFT the PRD            you, or an LLM writer agent
            │
            ▼
   2. ADVERSARY  ◄────────┐    Adversarial PRD Reviewer (this skill)
      hostile review      │    • 7-pass rubric: JTBD, traceability, scope, deps, metrics, $
      + Special Mandate   │    • Special Mandate: verify every repo claim, hunt loopholes
            │             │      (fake backend, no-data, skip-to-green, weakened assertions, forks)
            ▼             │
   3. REVISE the PRD ─────┘    writer fixes each FATAL/SERIOUS in the document;
      until verdict            reviewer re-reads to close → SHIP / SHIP WITH STIPULATIONS / DO NOT SHIP
            │                  (UNRESOLVED = an honest gap with an owner, never a fake green)
            ▼
   4. SPECFLOW WRITER          turn the hardened PRD into tickets:
            │                  Gherkin acceptance, data-testid selectors, contract refs, E2E journey files
            ▼
   5. BOARD AUDITOR + UPLIFT   fill missing SQL/RLS, TypeScript interfaces, invariants,
            │                  data-testid coverage → re-audit to compliance
            ▼
   6. PRE-FLIGHT SIMULATION    walk real personas through each ticket; a CRITICAL design gap
            │                  blocks the ticket (catches "this won't work for a real user" before code)
            ▼
   ✅ DEFENSIBLE GH EPICS + ISSUES
      buildable · honest · UNRESOLVED items flagged — not perfect, defensible
```

**Why this order:** the adversary makes the spec *correct and honest* up front; Specflow makes
it *enforceable* forever; the simulation proves it *works for a real user* before a line of
code. Skip step 2 and you get perfectly compliant tickets for the wrong thing.

---

## Install

**The adversary (this skill):**
```bash
# Codex CLI
git clone https://github.com/Hulupeep/adversarial-prd-reviewer ~/.codex/skills/adversarial-prd-reviewer
# Claude Code
git clone https://github.com/Hulupeep/adversarial-prd-reviewer ~/.claude/skills/adversarial-prd-reviewer
```

**Specflow (the companion for steps 4–6):** **https://github.com/Hulupeep/Specflow**
```bash
npx @colmbyrne/specflow init .   # contracts, hooks, writer/auditor/uplifter/simulator agents, CI gates
```

---

## Worked scenarios

Each is a realistic thing people build — usually **with a repo already in flight** — showing the
prompt, what the adversary catches, and what happens *after* the catch.

---

### Scenario A — B2B SaaS: "add team seats & billing"
**You already have:** a Next.js + Postgres + Stripe app with single-user subscriptions. You wrote
`docs/prd-team-billing.md` to add multi-seat plans.

**Claude Code prompt:**
```
Use the adversarial-prd-reviewer skill to harden this PRD before I write any tickets.
- Repo: this one. Read the files the PRD cites (Stripe integration, the subscriptions table) —
  verify my claims against the real code, don't trust the prose.
- Run the Special Mandate — this touches billing + a CI "test mode".
- PRD: @docs/prd-team-billing.md
Give me FATAL/SERIOUS first, then revise the PRD to close them and stamp a verdict.
```
**Codex prompt:**
```
$adversarial-prd-reviewer
Full review + Special Mandate on @docs/prd-team-billing.md against this repo. Open the real
Stripe + subscription files it references. Dueling mode: challenge, revise the doc, re-verify.
Output the Reality-Grounding Ledger, the Loophole Hunt, and a verdict.
```

**What it catches:**
- **FATAL — no willingness-to-pay chain.** "Add seats" never states the price, the
  `feature → behavior → revenue` mechanism, or who decides it. It's a feature spec, not a
  product spec.
- **SERIOUS — proration is unspecified.** Mid-cycle seat add/remove: two engineers would build
  two different billing behaviors. Untestable as written.
- **SERIOUS (Loophole: fake/no-data).** CI "test mode" uses a placeholder Stripe key, so the
  payment path never actually runs — green that proves nothing. Probe: does any test exercise a
  *real* Stripe test-clock, or just mock the webhook?
- **WEAK — banned language:** "seamlessly handle downgrades."

**After the catch:** the writer adds the pricing decision + exact proration rules, replaces the
fake key with a real Stripe **test-clock** journey, and names the empty-state ("seat limit
reached") UX. Verdict: **SHIP WITH STIPULATIONS** — `UNRESOLVED: Finance confirms the per-seat
price` (flagged on the epic, not hidden).
→ **Specflow writer** produces epic *Team Seats & Billing* with issues: seat add/remove,
proration, webhook idempotency, dunning, seat-limit UX — each with Gherkin + contract refs.
→ **Uplifter** adds the webhook RLS policy + an idempotency invariant.
→ **Simulation** walks "admin adds 3 seats mid-cycle" → surfaces a missing dunning-failure path →
new issue. **Result: a defensible epic with the price decision openly UNRESOLVED.**

---

### Scenario B — AI feature: "add an AI summarizer to our knowledge base"
**You already have:** a docs app with search. PRD: `docs/prd-ai-summary.md` to summarize a doc on
open.

**Claude prompt:**
```
adversarial-prd-reviewer skill, Special Mandate ON (this is an AI feature with a measurability
trap). Harden @docs/prd-ai-summary.md against this repo's retrieval code. I want the FATAL/SERIOUS
issues then a revised PRD.
```

**What it catches:**
- **FATAL — banned "leverage AI to intelligently summarize."** Demand input / process / output /
  failure handling, not an adjective.
- **FATAL — no eval oracle.** "Users get great summaries" is unmeasurable. There's no golden set,
  no measurement method → the success metric can't be evaluated without a human in the room.
- **SERIOUS (Loophole: tautology).** The proposed quality test has *the LLM judge its own
  summary* — the gate consumes its own output. Not validation.
- **SERIOUS — no failure path.** What happens when retrieval returns nothing? The spec implies the
  model answers anyway → a hallucination shipped as a feature.

**After the catch:** the writer defines input (the doc + retrieved context), process
(retrieve → summarize *with citations*), output (summary + cited sources), and the honest failure
path (empty/low-confidence retrieval → "not enough information," never a guessed answer). The eval
becomes a **labelled golden set + a human-rated sample** (an oracle the model can't grade itself
on). Verdict: **SHIP WITH STIPULATIONS** — `UNRESOLVED: who curates the golden set`.
→ **Specflow** epic *Doc Summarizer*: issues for retrieval, summarize-with-citations, the
empty-state path, and the **eval harness as a journey test**. → **Simulation** walks "summarize a
doc with no relevant content" and asserts the "not enough info" path fires. **Defensible: the
feature can't silently hallucinate, and its quality is measured against something real.**

---

### Scenario C — Infra: "replace our fake e2e backend with a real one"
**You already have:** an e2e suite that "passes" against a placeholder backend. PRD:
`docs/prd-real-e2e.md`.

**Codex prompt:**
```
$adversarial-prd-reviewer
Special Mandate REQUIRED — this is test-infrastructure honesty. Review @docs/prd-real-e2e.md
against the repo: open ci.yml, the supabase config, the seed files, and the test setup. Hunt
specifically for: fake URLs surviving, a health check that proves liveness but not seeded data,
skip-to-green, and any second seed source. Revise to close, then verdict.
```

**What it catches:**
- **FATAL — the seed creates users but zero auth identities.** A real backend that nobody can log
  into; every login-gated test still fails/skips. (Verified by reading the seed, not the prose.)
- **FATAL (Loophole: no-data).** The "backend ready" check pings `/health` — proves the API is
  *up*, not that it has *seeded data*. The classic "spins up = done."
- **SERIOUS — a fake placeholder URL** survives in one CI job.
- **SERIOUS (Loophole: fork).** An inline test fixture seeds its own users — a second source of
  truth that will drift from the canonical seed.

**After the catch:** add auth-user seeding (both tables), a health check that asserts a *real
login + a seed-row count*, delete the fake URL, retire the fixture (reuse the one seed). Verdict:
**SHIP WITH STIPULATIONS** — `UNRESOLVED: pin the CLI version`.
→ tickets → uplift (the RLS + the health-check script) → simulation. **The e2e suite now tells the
truth — including the uncomfortable truth that some features were broken all along.**

---

### Scenario D — Greenfield: "scheduling app for physiotherapy clinics"
**You have:** an empty repo and an idea. PRD: a one-pager.

**Claude prompt:**
```
adversarial-prd-reviewer skill. No repo yet — this is idea-stage, so run the rubric hard on JTBD,
scope boundaries, and imaginary users. Harden @prd.md until it's buildable, then verdict.
```

**What it catches:**
- **FATAL: NO JTBD.** "Clinics need scheduling" — *which* role, and what can't they do today?
  No named user with a real unmet need.
- **SERIOUS — load-bearing scope unmentioned:** no-shows, cancellations, double-booking,
  waitlists. The dangerous third category (not in-scope, not out-of-scope — just absent).
- **WEAK — "comprehensive scheduling"** (unbounded scope).

**After the catch:** the writer names the persona (front-desk admin at a 2–5 therapist clinic;
today they juggle paper + phone tag and double-book), enumerates scope (book / reschedule /
cancel / no-show / waitlist) and explicitly puts *insurance billing* out of scope. Verdict:
**SHIP.** → A defensible MVP epic with real, bounded issues — instead of "build a comprehensive
scheduling system" that means nothing.

---

## What "defensible, not perfect" means

The pipeline does **not** promise a perfect spec or an all-green board. It promises **honesty**:

- Every requirement survived a hostile review (two-engineer-buildable, testable, scoped).
- Every concrete claim about the repo was checked against the real files.
- Every silent-lie loophole was hunted, not assumed absent.
- Gaps that need a human (a price, an owner, a date, an evidence source) are **flagged
  `UNRESOLVED` on the epic/issues** — visible, not buried.
- The simulation proved each ticket works for a real persona before code.

A defensible epic can have open questions. What it cannot have is a hidden lie wearing a green
checkmark.

---

## Copy-paste prompt templates

**Step 2 — Adversary (Codex):**
```
$adversarial-prd-reviewer
Full rubric review of <PRD path/paste> against this repo. Open every file/port/line the PRD
cites and verify it — don't trust my summary. Run the Special Mandate if this touches CI gates,
coverage, test infra, seed/fixtures, reconciliation, attestation, or drift.
Dueling mode: lead with the FATAL→SERIOUS→WEAK issue list, then revise the PRD text to close each,
re-verify your own fixes, mark human-only gaps UNRESOLVED, and stamp SHIP / SHIP WITH STIPULATIONS
/ DO NOT SHIP. Output the Reality-Grounding Ledger and the Loophole Hunt table.
```

**Step 2 — Adversary (Claude Code):**
```
Use the adversarial-prd-reviewer skill to harden <@PRD> before I write tickets.
Repo: this one — read the files it references and verify my claims against reality.
Run the Special Mandate (this touches <area>). Give me FATAL/SERIOUS first, then revise the PRD
to close them, then a verdict with any UNRESOLVED items named.
```

**Steps 4–6 — Specflow (after the PRD ships):**
```
The PRD is hardened (SHIP / SHIP WITH STIPULATIONS). Now:
1. specflow-writer: turn it into a GitHub epic + issues (Gherkin acceptance, data-testids,
   contract refs, E2E journey files).
2. board-auditor + specflow-uplifter: uplift each ticket to full compliance, then re-audit.
3. pre-flight-simulator: walk the key personas through each ticket; block on CRITICAL design gaps.
Carry the PRD's UNRESOLVED items onto the epic so they stay visible.
```
(Install: https://github.com/Hulupeep/Specflow)
