# AI Skill Template — Method Summary

This project implements a **Documentation-Driven Development (DDD)** workflow powered by two AI skills and a structured document system. The core idea: the document is the single source of truth — AI writes it, humans review it, then AI implements from it.

---

## The Development Loop

```
Human proposes requirement (brief description or idea)
        ↓
[doc-maintain] AI drafts the implementation document (FXX / RXX / BXX)
        ↓
Human reviews and approves the document
        ↓
[tdd-development] AI reads document → writes failing test → implements code → verifies criteria → updates document
        ↓
[doc-maintain] syncs module docs to reflect new implementation state
```

This loop places **humans in the role of planner and reviewer**, not executor. The quality of the output depends on the quality of the document — which is why human review of the draft doc is the critical gate before any code is written.

---

## The Two Document Categories

### Category 1 — Implementation Documents (`documents/implements/`)

**Purpose:** Record and clarify the scope of each unit of work given to AI for implementation.

These are created *before* coding begins and updated *after* implementation completes. They serve as the AI's authoritative prompt — replacing vague chat instructions with structured, verifiable specifications.

| Type | Prefix | Template | When to use |
|---|---|---|---|
| Feature spec | `FXX` | `F00-功能需求書模板.md` | New functionality |
| Refactor spec | `RXX` | `R00-重構任務模板.md` | Structural improvement without behavior change |
| Bug fix spec | `BXX` | `B00-Bug修正模板.md` | Reproducing and fixing a defect |

Each document includes:
- User story and feature background
- Acceptance criteria in Given/When/Then format
- Test scenarios table (ID, Given, When, Then, Priority)
- Implementation notes and constraints
- An **Implementation Record** section filled in by AI after coding (status, test evidence, files changed, gaps)

All three types share the same TDD discipline: **Red → Green → Refactor → Sync Document**.

---

### Category 2 — Module Documents (`documents/modules/`)

**Purpose:** Reflect the *current* state of the codebase at a high level, so a new engineer can quickly understand the system without reading every file.

These are **not** task documents — they are living reference docs that must stay synchronized with the actual implementation at all times.

Key characteristics:
- **High-level only.** Focus on responsibilities, boundaries, data flow, and architecture — not line-by-line code details.
- **Always in sync with code.** When code and module doc disagree, the code is truth. The doc must be corrected.
- **Engineer-oriented.** Written so a new team member can orient themselves quickly — understand what a module does, where it lives, what it depends on, and what its known limitations are.
- **No future-tense fiction.** Do not describe unimplemented behavior as if it exists.

Good module doc sections: Purpose · Key files · Main responsibilities · Data/state flow · External dependencies · Known limitations · Related FXX/RXX/BXX documents.

---

## The Two Skills

### `doc-maintain`

**Used for:** Creating and maintaining both document categories.

**In the development loop — before coding:**
1. Reads the human's requirement description.
2. Finds the next available FXX/RXX/BXX number and reads the matching template.
3. Checks `documents/modules/` and the codebase for relevant context.
4. Drafts a complete implementation document for human review.

**In the development loop — after coding:**
1. Updates the FXX/RXX/BXX Implementation Record with final results.
2. Audits affected module docs against the new implementation.
3. Updates or flags module docs that are now stale.

---

### `tdd-development`

**Used for:** All production code changes, driven by an approved implementation document.

**Sequence (non-negotiable):**
1. Read and understand the FXX/RXX/BXX document — this is the only authoritative source of requirements.
2. Extract acceptance criteria and test cases from the document.
3. Write the smallest failing test (red phase) — must confirm failure before proceeding.
4. Implement the minimum production code to pass (green phase).
5. Verify every acceptance criterion is covered by tests.
6. Write the Implementation Record back into the document.

Never starts from a vague chat instruction alone. If no implementation document exists, stops and requests one.

---

## Key Design Principles

- **AI drafts, humans approve.** The document creation step is a collaboration checkpoint, not a formality.
- **No code without an approved document.** `tdd-development` treats the document as its only prompt.
- **No success without test evidence.** A feature is not complete unless tests ran and passed.
- **Module docs mirror reality.** Stale module docs are treated as bugs by `doc-maintain`.
- **Minimum viable change.** Both skills enforce smallest-test and smallest-implementation discipline.
- **Assumptions are surfaced, not buried.** Ambiguity in a document is flagged before implementation, and any assumptions made are recorded in the Implementation Record.
