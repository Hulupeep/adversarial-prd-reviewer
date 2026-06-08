# Adversarial PRD Reviewer

A portable **agent skill** that makes a PRD (or any spec — RFC, design doc, ticket set,
acceptance criteria, API contract) survive a *hostile* product review **before** engineering
spends a sprint on it. Works with **Codex CLI** and **Claude Code**.

It does two jobs the usual "looks good 👍" review never does:

1. **Destroys ambiguity** — interrogates every requirement until two engineers who never spoke
   to the author would build the *same* thing.
2. **Destroys silent lies** — actively hunts for the plausible-but-dishonest shortcut: a gate
   that's gamed, a backend that's fake, a test that skips itself green, a "fact" about the repo
   that isn't true.

> A spec can be perfectly *buildable* and still be a silent lie with a green checkmark on it.
> This skill is built to catch both.

**👉 New here? Read [PIPELINE.md](./PIPELINE.md)** — the full practice with *worked examples*
(real products, copy-paste prompts for **Codex** and **Claude**), what happens *after* the
adversary catches something, and how it feeds **[Specflow](https://github.com/Hulupeep/Specflow)**
to end at GitHub epics & issues that are **defensible, not perfect**.

---

## How it works

The skill is the loop **Agent A writes → Agent B attacks → A revises in the document → B
re-reads and closes or reopens**, repeated until every issue is resolved or explicitly marked
unresolved. A single model can play both roles by switching hats.

### Operating modes
| Mode | What it does |
|------|--------------|
| **Review an existing PRD** | Run all rubric passes, lead with a severity-ordered issue list, maintain issue counts to a verdict. |
| **Dueling writer / reviewer** (primary) | A drafts, B challenges with the rubric, round after round. B never edits; A never closes its own issue. |
| **Improve a PRD** | Produce revised spec language for each accepted fix + a changelog. |
| **Special-Mandate** | For honesty/integrity-critical specs: adds reality-grounding + a loophole hunt on top of the rubric. |

### The rubric (`references/review-rubric.md`)
Seven mandatory passes before any verdict:
1. **Jobs-to-be-Done coherence** — named users with real unmet needs (else `FATAL: NO JTBD`).
2. **Requirement → implementation traceability** — "could two engineers build the same thing?"
3. **Two-engineer test** — every clarifying-question-needed point is a blocker.
4. **Scope boundary** — in / explicitly-out / dangerously-unmentioned.
5. **Dependency & ordering** — nothing assumed ready without evidence.
6. **Success-metrics audit** — numbers, direction, window, measurement method — or rejected.
7. **Revenue / willingness-to-pay** — the full `feature → behavior → mechanism → amount` chain.

Plus a **banned-language scan** (no "robust", "seamless", "configurable", "etc.", passive
"should be sent"), a **Ramen Test** (describe it without jargon; what do people use today?),
**Specflow / multicheck readiness**, and a final verdict: **`SHIP` / `SHIP WITH STIPULATIONS`
/ `DO NOT SHIP`** with maintained issue counts (`FATAL` > `SERIOUS` > `WEAK`).

### The Special Mandate (`references/adversarymandate.md`)
The rubric proves a spec is *buildable*. The Mandate proves it is *honest and real*, on two
extra axes:

- **Axis A — Reality-Grounding:** the reviewer opens **every concrete repo claim** the author
  makes (file contents, line numbers, ports, command flags, "X is the only source of Y") and
  records verified-or-not in a **Reality-Grounding Ledger**. A false load-bearing claim is
  `FATAL`/`SERIOUS` — engineering must not build on a fiction.
- **Axis B — The Loophole Hunt:** for each known silent-lie smell, the reviewer *tries to find
  it surviving* (not confirm its absence):

  | Smell | What it is |
  |-------|------------|
  | Fake / placeholder backend | a `test.*.co` / dummy key survives in a runtime path |
  | No-data / empty-resource | the dependency spins up but is **empty**, so tests skip/trivially pass — "spins up = done" |
  | Skip-to-green | `test.skip(true,…)` hides a red with no tracked reason |
  | Weaken-assertion | a "fix" passes by asserting *less* (`toBe`→`toBeTruthy`, swallowed `.catch`) |
  | Duplicate source of truth | a second seed/fixture/config/explainer that will drift |
  | Gate-gaming | lowering a threshold to pass, with no honest ratchet + reason |
  | "All green" as success | success defined as "suite passes" instead of "every red is a filed bug" |
  | Mock-as-real | a mock substitutes for the real stack and counts as validation |
  | Tautology | a gate consumes the answer it's supposed to check as its own input |

  **No `SHIP` while any loophole-class issue is unresolved.**

---

## What the data shows — real catches

These are the kinds of issues the Special Mandate caught in practice that would otherwise have
shipped as a **confident green that lies**. Each was a `FATAL`/`SERIOUS` found *before* merge:

- **A "real backend" with nobody home.** A spec to replace a fake test backend with a real,
  seeded one — but the seed created employee rows and **zero auth users**. The backend would
  have had data that *nobody could log in to*; every login-gated test would still fail or skip.
  Verified live: `auth.users = 0` against the canonical seed. → fixed before merge.
- **A health-check that proved the wrong thing.** "Backend is ready" was a ping to
  `/auth/v1/health` — which proves the API is *up*, not that it has *seeded data*. The classic
  "spins up = done" loophole. → readiness changed to assert a real login + seed-row count.
- **A fake URL hiding in plain sight.** A placeholder backend URL (`test.<service>.co`) left in
  the CI path so the suite "ran" against nothing.
- **A second source of truth.** An inline test fixture quietly seeding its own org/users — a
  duplicate of the canonical seed that would drift the moment either changed.
- **A guard that punished correctness.** A "no fake URL" grep that *false-positived the
  legitimate build job*, i.e. would have failed CI exactly when the work was done right.
- **A gate enforcing nothing.** An 80% coverage threshold on a codebase at ~2.4% — permanently
  red since long before, so everyone ignored it. A gate that's always red is itself a silent
  lie. → replaced with an honest floor-at-current-reality + ratchet-up plan.
- **"All green" smuggled into the metrics.** A spec whose success criterion was "the suite
  passes" rather than "every remaining red is a filed bug or tracked skip." → rewritten so the
  honest bar is *truthful*, not *green*.

The lesson behind all of them: **"it ran / it's green / the file says so" is not evidence.**
The Mandate forces the reviewer to verify against reality and to refuse the cheap green.

---

## Files

| File | What it is |
|------|------------|
| `PIPELINE.md` | The full end-to-end practice + worked examples (prompts, catches, Specflow handoff). |
| `SKILL.md` | The skill entry point: operating modes + required references. |
| `references/review-rubric.md` | The 7-pass rubric, severities, banned language, Ramen test, Specflow/multicheck, verdicts. |
| `references/dueling-protocol.md` | The A-writes / B-attacks runbook: roles, round cycle, issue ledger, termination. |
| `references/adversarymandate.md` | The Special-Mandate supplement: reality-grounding + loophole hunt + the two output tables. |
| `agents/openai.yaml` | Codex agent interface manifest (display name, default prompt). |

---

## Install

The skill is a directory of Markdown — drop it where your agent discovers skills.

### Codex CLI
```bash
git clone https://github.com/Hulupeep/adversarial-prd-reviewer \
  ~/.codex/skills/adversarial-prd-reviewer
```
Then invoke it in a Codex session:
```
Use $adversarial-prd-reviewer to interrogate this PRD and force it into a defensible, buildable spec.
```

### Claude Code
```bash
git clone https://github.com/Hulupeep/adversarial-prd-reviewer \
  ~/.claude/skills/adversarial-prd-reviewer
```
Then reference it by name or ask for an **adversarial PRD review** of your spec. For
honesty-critical work (CI gates, coverage, test infra, seed/fixtures, reconciliation,
attestation, drift), explicitly ask it to **run the Special Mandate** so it does the
reality-grounding + loophole hunt, not just the rubric.

> **Updates:** `git -C ~/.codex/skills/adversarial-prd-reviewer pull` (or the `.claude` path).

---

## How to use it well

1. **Give it the full spec, not a summary** — the reviewer must read the whole thing (Phase 0).
2. **Point it at the real repo** — for the Special Mandate to ground claims, it needs to open
   the actual files the PRD cites.
3. **Make A resolve in the document** — an answer in chat doesn't count; it must be written into
   the spec, then B re-reads to close.
4. **Read the verdict honestly** — `SHIP WITH STIPULATIONS` means the remaining gaps are
   human-only (provision a secret, name an owner) and explicitly `UNRESOLVED` — never that a
   design still has a way to lie.

---

## Extending it

The Mandate is meant to grow. When a new dishonesty pattern is caught in the wild, **add a row
to the loophole taxonomy** in `references/adversarymandate.md` (how it hides / the probe) so it's
caught by the rubric next time, not by luck. A new honesty invariant in an ADR should get a
corresponding probe here. The anti-drift tool must not itself drift.

---

## Addendum: the full execution pipeline (adversary + Specflow + ruflo)

This skill is **Gate A** of a larger loop that turns a rough idea into merged code you can trust —
even when a multi-agent swarm (e.g. [ruflo](https://github.com/ruvnet/ruflo) / claude-flow) does
the building. The principle:

> **The swarm is muscle *inside* a phase. The gates live *between* phases, and every gate is owned
> *outside* the swarm. Throughput is delegated; trust never is.**

```
[0] DISCOVER ─► [1] PRD ─[GATE A]─► [2] TICKETS ─[GATE B][GATE B.5]─► [3] BUILD ─[GATE C]─► merged
  human+agent     dueling   adversary  specflow      audit+    pre-flight   ruflo      specflow
  vs real         writer/   verdict    writer        closure   simulation   swarm      CI
  artifact        adversary (HARD)     (ruflo //)    validator (own gate)  (ruflo //)  (unfakeable)
```

| Gate | After | What it is | Type |
|------|-------|------------|------|
| **A** | the PRD | The adversary verdict must be `SHIP` / `SHIP WITH STIPULATIONS`, written to a **committed `verdict` artifact**. The controller refuses to spawn ticket-writing unless that artifact says SHIP. | **HARD** |
| **B** | tickets | `specflow audit` + closure validator — every requirement → journey → test → issue, no orphans, no duplicate IDs. | soft (controller) |
| **B.5** | tickets | **Pre-flight simulation** — walk real personas through each ticket; a CRITICAL design gap blocks. *Its own gate.* | soft (controller) |
| **C** | build | **Specflow CI** — contract tests + journey tests against a *real seeded backend* + anti-pattern audit + coverage ratchet. Runs in CI under branch protection: a violation **cannot merge**. | **HARD (unfakeable)** |

**Who does what:** *Discover* — human + agent vs the real artifact (no swarm). *PRD* — dueling
writer/adversary, strong models (quality > parallelism). *Tickets* — specflow-writer fanned out by
ruflo. *Build* — ruflo swarm: one implementer per ticket in worktrees, multi-model cost routing
(cheap models for boilerplate, strong for hard), shared memory, kill-switch dashboard.

**Why it holds:** Gate A is a *single hostile critic*, not the swarm's self-consensus; Gate C is
*CI*, not an agent's opinion. **Soft front (controller-enforced) + hard backstop (branch-protected
CI)** means even a gamed soft gate — or a fooled adversary — can't merge a contract violation. And
because Gate C's journeys run against a real seeded backend, *green-but-broken* can't pass.

**ruflo's role is throughput, not trust** — use it where work is genuinely parallel (tickets,
build, triage) for multi-model cost + memory + observability; keep the trust-critical moments
(Gate A, Gate C) outside the swarm. **No fork needed** — the adversary is a runtime skill the
critic invokes; Specflow is a CLI + CI the swarm calls.

Companion: **[Specflow](https://github.com/Hulupeep/Specflow)** (Gates B / B.5 / C) · full
non-swarm walkthrough in [PIPELINE.md](./PIPELINE.md).

---

## License

MIT — see [`LICENSE`](./LICENSE).
