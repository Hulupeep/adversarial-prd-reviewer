# Adversary Special-Mandate Mode

> Supplement to `review-rubric.md`. The rubric proves a PRD is **buildable**
> (coherent, testable, traceable, two-engineer-safe). The Special Mandate proves it
> is **honest and real** — that it does not encode a plausible-but-lying solution and
> is not built on claims that don't match the actual repo.
>
> A PRD can pass all seven rubric passes and still ship a silent lie: a gate that's
> gamed, a backend that's fake, a test that skips itself green, a second source of
> truth that will drift. The standard rubric checks *internal* quality. The Mandate
> adds two *external* axes: **does it match reality**, and **does it refuse the
> dishonest shortcut**.

## When to invoke
Turn on Special-Mandate mode when ANY hold:
- The PRD operationalizes an ADR with **honesty/integrity invariants** ("no silent
  lies", "derived not asserted", "real not mocked", "ratchet not lower").
- The danger is **not vagueness but a plausible cheat** — a green that lies. Anything
  touching CI gates, coverage thresholds, test infra, seed/fixtures,
  reconciliation/validation, drift, attestation, golden-masters.
- The PRD makes many **concrete, checkable claims** about the repo (paths, line
  numbers, ports, "file X does Y", command flags, "the seed creates Z").

If none hold, the standard rubric suffices. If they hold, the rubric is **necessary
but not sufficient** — run both.

## Axis A — Reality-Grounding: close every concrete claim against what's real
A spec built on a false premise is worse than a vague one: it's confidently wrong.
The author believes their own description of the repo. **You do not.** Verify it.

1. Extract every concrete claim about the codebase. Types: file/path contents;
   line-number citations; config values; command/flag correctness; behavioral claims
   about existing code; "X is the canonical/only source of Y".
2. **Open the real file / run the real grep for each.** Do not trust the prose or the
   author's summary of the file — read the file.
3. Record each in a **Reality-Grounding Ledger** (see Output). A claim that does not
   hold is at least `SERIOUS` — engineering will build on a fiction.
4. Prioritise claims load-bearing for an honesty invariant (e.g. "the health check
   proves seeded data" — does it, or does it only prove the endpoint is up?).

## Axis B — The Loophole Hunt: try to FIND each silent-lie smell
For each smell, your job is not to confirm absence — it's to **try to find it
surviving**. Assume the author, under deadline, reached for the cheap green.

| Smell | How it hides | The probe |
|---|---|---|
| **Fake / placeholder backend** | test/staging URL, dummy key, `example.com`, `test.*.co`, empty `localhost` | grep PRD + cited config for placeholder URLs/keys; confirm every runtime path hits a *real provisioned* dependency |
| **Empty-resource / no-data** | resource provisioned but **empty**, so tests skip/trivially pass — "spins up, therefore done" | does readiness prove **data exists** (a seeded row count), not just that the endpoint answers? API-ping only = loophole open |
| **Skip-to-green** | `test.skip(true,…)`/`fixme` hides red with no tracked reason | is every skip tied to an issue? is free-text skip forbidden? can a failure be silenced by skipping? |
| **Weaken-assertion** | `toBe(x)`→`toBeTruthy()`, `.catch` swallow, loosened regex | does any "fix" pass by asserting *less*? real assertions only |
| **Duplicate source of truth (DRY/fork)** | a 2nd seed/fixture/config/explainer that drifts | is there exactly ONE source? does it reuse it or fork an N-th copy? forks are silent lies on a delay |
| **Gate gaming** | lower a threshold to pass, no honest ratchet + reason | honest floor-at-current-reality + ratchet-up plan, or just "make it pass"? |
| **"All green" as success** | success = "suite passes" not "every red is a filed bug" | is "all green" smuggled into Goals/Metrics? honest bar is *truthful*, not *green* |
| **Mock-as-real** | a mock substitutes for the real stack and counts as validation | tests the real dependency, or a mock that drifts? |
| **Tautology** | a gate consumes the answer as its own input | expected value derived independently, or fed from the thing under test? |

**Sharpen from the ADR:** for each honesty invariant in the governing ADR, write a
probe of the form *"find a place in the PRD where this invariant is violated or left a
gap."* Every invariant → one adversarial probe.

## Severity rules specific to the Mandate
- Surviving loophole an invariant forbids → **FATAL** (the PRD defeats its purpose).
- False load-bearing reality claim → **FATAL/SERIOUS**.
- "All green" present as a success criterion → **SERIOUS** until rewritten.
- Cosmetic wrong line-number / naming mismatch → **WEAK**.
- **No `SHIP` while any loophole-class issue is unresolved.** `SHIP WITH STIPULATIONS`
  only if the remainder is human-only (provision a secret, name an owner), marked
  UNRESOLVED — never if the gap is "the design still has a way to lie."

## Required output additions
```markdown
## Reality-Grounding Ledger
| # | PRD claim | Location | Verified against | Holds? | Evidence / finding |
|---|-----------|----------|------------------|--------|--------------------|
| 1 | "ci.yml:106-108 set fake env" | §Provisioning | .github/workflows/ci.yml | ✅ | sets test.supabase.co |
| 2 | "API port 54331" | §Provisioning | supabase/config.toml | ✅ | [api] port = 54331 |
| 3 | "health check proves seeded data" | §Provisioning | (design) | ⚠️ | only pings /auth/v1/health — proves no seed row → LOOPHOLE ISS-n |

## Loophole Hunt
| Smell | Found? | Location | Severity | Required fix |
|-------|--------|----------|----------|--------------|
| Empty-resource / no-data | FOUND | health check | FATAL | assert a seed-row count, not just API liveness |
| DRY / seed fork | FOUND | tests/global-setup.ts inline fixture | SERIOUS | retire the fixture; reuse canonical seed |

## Honest-Success-Criterion Check
- "All green" appears as a goal/metric? [yes/no] → if yes, SERIOUS until rewritten.
- Stated bar is "every remaining red is a filed bug or tracked fixme"? [present/absent]
```

## The meta-principle
Standard rubric: *"Could two engineers build the same thing from this?"*
Special Mandate: *"If they build exactly this, will it tell the truth — and is every
concrete thing it claims actually true?"* A spec can be perfectly buildable and still
be a silent lie with a green checkmark on it.

## Improving the Mandate over time
- New dishonesty pattern caught in the wild → **add a taxonomy row** (how it hides /
  the probe). Don't let it live in one reviewer's head.
- A class of reality-claim that repeatedly proves wrong → **promote to a mandatory
  ledger row**.
- Keep DRY with the ADRs: a new honesty invariant must get a corresponding probe here.
  The Mandate that documents anti-drift must not itself drift.

## Provenance
Distilled from the special-mandate briefs used repeatedly in practice (e.g. the
decision-integrity and e2e-real-backend reviews), where the adversary caught: a fake
Supabase URL surviving in CI; a health-check that proved liveness but not seeded data;
a seed with zero `auth.users` (real backend, nobody could log in); an inline
`global-setup.ts` fixture as a second seed source; a forbidden-URL grep that
false-positived the legitimate build job; and "all green" smuggled in as success.
Each is now a named smell above so it is caught by the rubric, not by luck.
