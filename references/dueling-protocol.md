# Dueling Writer / Reviewer Protocol

This is the runbook for the two-agent format: **Agent A drafts, Agent B attacks.**
As A writes, B challenges. The loop runs until every challenge is resolved or
explicitly marked unresolved. Use this for a PRD or any spec (RFC, design doc,
ticket set, acceptance criteria, API contract).

## Roles

### Agent A — The Writer

- Owns the document. A is the only agent that edits spec text.
- Drafts requirements, assumptions, acceptance criteria, scope, dependencies,
  metrics, and revenue logic.
- Responds to each of B's challenges by **changing the document**, not by arguing
  in chat. An answer that is not written into the spec does not count.
- May mark a gap `DEFERRED` or `OUT OF SCOPE` instead of resolving it — but must
  write that decision and its owner into the spec.

### Agent B — The Adversarial Reviewer

- Owns the rubric (`references/review-rubric.md`). B never edits the document.
- Runs the seven passes, the banned-language scan, the Ramen Test, and the
  two-engineer test against whatever A has produced so far.
- Emits challenges, each with: location, one-sentence problem, the exact question
  that must be answered, and the prohibition on vague answers.
- Verifies fixes by re-reading A's revised text — not by trusting A's claim that
  it was fixed.
- Maintains the issue ledger and is the only agent that closes an issue.

Single-operator note: when one model plays both parts, switch hats explicitly.
Label every turn `A:` or `B:` and never let A close its own issue — B must
re-read the revised text first.

## Round Protocol

Work proceeds in numbered rounds. Each round is one full A→B→A cycle.

```
Round N
  A:  Draft or revise a section (or the whole spec on round 1).
  B:  Run rubric passes on the current text.
      Emit new challenges. Re-check open challenges. Update the ledger.
  A:  Edit the document to resolve each open challenge, or mark it
      DEFERRED / OUT OF SCOPE with an owner.
  B:  Re-read the edited text. Close what is fixed. Reopen what is not.
```

Rules:

1. B challenges only the current text. No challenging a draft A has already revised.
2. A resolves in the document. Chat-only answers are rejected and re-asked.
3. An issue that survives two full rounds is escalated (see Escalation).
4. New challenges may appear in any round as the spec grows — fixing one section
   often exposes a gap in another. The loop is not done just because round 1's
   list is clear.

## The Issue Ledger

B maintains and prints this every round so progress is visible:

```
ISSUE LEDGER — Round N
  Total identified:        [N]
  Open (FATAL):            [list of ids]
  Open (SERIOUS):          [list of ids]
  Open (WEAK):             [list of ids]
  Resolved in document:    [count]
  Deferred / out of scope: [count, each with owner]
  Verdict (current):       DO NOT SHIP | SHIP WITH STIPULATIONS | SHIP
```

Each issue id is stable across rounds (e.g. `B-003`) so A and B refer to the
same thing. Severity comes from the rubric: FATAL > SERIOUS > WEAK.

## Challenge Format (what B emits)

```
[B-007] SERIOUS — §3.2 Notifications
Problem:  "Notifications should be sent" hides the actor and trigger.
Question: Which service sends what payload, on which event, to which recipient,
          and what happens when delivery fails?
No vague answers: name the actor, the event, the channel, and the failure path.
```

## Resolution Format (what A emits)

```
[B-007] RESOLVED in §3.2
Changed text: "On leave_request.approved, the LeaveService publishes a
push notification to the requester's registered devices; on delivery failure
it retries 3× then writes a digest row read by the daily email job."
```

B then re-reads §3.2 and either closes `[B-007]` or reopens it with a one-sentence reason.

## Escalation

If an issue survives two full rounds without resolution, B posts:

> This has taken two rounds. The spec cannot ship with this gap. Give me the
> final language now or I am flagging `[B-id]` as UNRESOLVED.

A then either writes final language or accepts the `UNRESOLVED` mark, which forces
the verdict down (any open FATAL ⇒ `DO NOT SHIP`).

## Termination

The loop stops only when one of these holds:

- **SHIP** — zero open FATAL or SERIOUS issues; WEAK issues all resolved or
  explicitly accepted by the operator.
- **SHIP WITH STIPULATIONS** — zero open FATAL; remaining issues are written into
  the spec as named, owned, dated stipulations.
- **DO NOT SHIP** — one or more FATAL issues remain UNRESOLVED.

Before declaring SHIP or SHIP WITH STIPULATIONS, B runs Final Verification from the
rubric (full re-read, banned-language scan, two-engineer re-test on changed
sections, fix-present check on every closed issue).

## Producing the Final Prompt

If the user asks for a reusable prompt that runs this workflow with two subagents,
emit a prompt that contains, in this order:

1. The two role definitions (A writer, B reviewer) verbatim in spirit.
2. The round protocol and the rule that A edits, B never edits.
3. The challenge / resolution formats.
4. The issue ledger and the requirement to print it each round.
5. Banned-language enforcement (point B at the rubric).
6. The escalation rule and the three termination verdicts.
7. The Final Verification gate before any SHIP verdict.
