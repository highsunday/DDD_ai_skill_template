---
name: grill-me
description: Interview the user relentlessly about a plan, design, or bug until reaching shared understanding, walking each branch of the decision tree and resolving dependencies one-by-one. Tracks which branches are resolved vs still open, gives a recommended answer for every question, and stops only when no open branch can still change the plan. Use when the user wants to stress-test a plan, get grilled on a design, clarify queue items, or pin down a bug's symptom / reproduction / done-condition.
---

# grill-me

Interview the user relentlessly about every aspect of this plan, design, or bug until you reach a **shared, decision-complete understanding**. Walk down each branch of the decision tree, resolving dependencies between decisions one-by-one. The goal is not to collect answers — it is to eliminate every unknown that could still change what gets built.

## How to run the interview

- **One question at a time.** Never batch. Each answer reshapes the tree and may make later questions moot or raise new ones.
- **Always recommend an answer.** For every question, state your recommended choice and the one-line reasoning, so the user can just confirm. Phrase it so a "yes" is a complete answer.
- **Explore the codebase instead of asking** whenever a question can be answered from the code, docs (`CONTEXT.md`, ADRs, existing modules), git history, or logs. Only ask the human what only the human knows: intent, priorities, trade-offs, external constraints.
- **Follow dependencies, not a checklist.** Resolve the decision that unblocks the most downstream questions first. When an answer opens a new branch, descend into it before moving sideways.
- **Push on vague answers.** "It should be fast", "handle errors gracefully", "the usual" are not answers — convert them into a number, a concrete behavior, or a named case before moving on.

## Track the tree out loud

Maintain a short running ledger so neither side loses the thread. After each answer, keep this current:

- **Resolved** — decisions now locked, with the chosen answer.
- **Open** — branches still unanswered, ordered by how much they constrain the rest.
- **Assumptions** — things you're proceeding on without explicit confirmation (flag them so the user can veto).

Surface it compactly (a few lines) whenever the tree shifts meaningfully, not after every single question.

## When to stop

Stop when **every open branch has been resolved or explicitly deferred**, and no remaining unknown could still change the plan. Then give a tight summary of the locked decisions and the assumptions you're carrying. Do not declare done while an open branch could still flip a major decision — that's the whole point of the grilling.

## Mode: bug intake (used by `ddd-debug-trace`)

When grilling to open a bug investigation, drive down three trunks until each is precise enough for an AI to act on autonomously:

1. **Symptom** — actual vs expected behavior, plus the **error fingerprint** (exact message / stack trace / anomalous value) used later to recognize recurrence.
2. **Reproduction** — precise enough to replay: environment, versions, config, input, command. Pin down **frequency**: stable / intermittent (Y in X) / one-off — and for intermittent, hunt for the trigger condition.
3. **Done condition** — a concrete, measurable metric (pass rate, latency, error gone), and always its **current baseline value** — without a baseline you can't recognize partial improvement.

Treat unmeasured or hand-wavy answers as open branches: keep pushing until the symptom, the repro, and the done-metric-with-baseline are all nailed down.
