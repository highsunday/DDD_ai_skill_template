# AI Skill Template — Enhanced Methodology Summary (DDD × Pocock)

This project builds on the original DDD workflow by integrating Matt Pocock's skill library, strengthening four failure nodes. The core philosophy remains unchanged: documents are the single source of truth, AI drafts, the user reviews, then AI implements accordingly. The added skills are inserted into the cycle as **plug-in points**, not as replacements for the skeleton.

---

## Unified Development Cycle

```
User submits a requirement
        ↓
[ddd-start]  Detect requirement type, route to the correct entry point
        ↓
  ┌─ Vague idea ──→ [grill-me]        Deep-dive to surface real requirements
  ├─ Involves architecture ──→ [grill-with-docs]  Validate against existing domain model
  ├─ Large multi-phase change ──────────────────────────────────────────────────────────┐
  │                                                                                     ↓
  │                                                    [ddd-plan]  Draft PXX plan doc
  │                                                                                     │
  │                                                        User reviews and approves PXX
  │                                                                                     │
  │                                             Execute phase by phase (each phase follows the flow below)
  │                                                                                     │
  ├─ Long-running work queue ──────────────────────────────────────────────────────────┐
  │                                                                                     ↓
  │                                                    [ddd-queue]  Create QXX queue
  │                                                                                     │
  │                            Centralized [grill-me] to clarify all items
  │                                                                                     │
  │                     Each item opens a new session to run ddd-start/doc/tdd
  │                                                                                     │
  └─ Spec already exists ──→ Proceed directly to document step ←───────────────────────┘
        ↓
[ddd-doc]  Draft FXX / RXX / BXX document (if from PXX, draft per the plan's suggested type)
        │
        ├─ For RXX document → [zoom-out]  Confirm refactor scope boundary
        └─ Informal description → [to-prd]    Convert to structured draft
        ↓
User reviews and approves document
        ↓
[ddd-tdd]  Red → Green → Acceptance criteria
        │
        └─ Unexpected test failure → [diagnose]  Structured debugging → results written back to FXX
                              └─ Hard bug requiring multiple experiments or sessions
                                 → [ddd-debug-trace]  Persist the investigation in documents/bugs/
        ↓
[ddd-doc]  Sync FXX records + module documents
        │     If from PXX: back-fill associated document numbers, update phase status
        │
        └─ Architecture technical debt found → [improve-codebase-architecture]  Output RXX candidates
```

This cycle **places the user in the role of planner and reviewer**. Pocock skills provide structured recovery paths at each failure node, rather than after-the-fact remediation.

---

## Three Document Categories

### Category Zero — Planning Documents (`documents/planning/`)

**Purpose:** Break large changes that cannot be completed in one pass into multiple ordered phases, serving as the top-level planning basis for subsequent F/R/B documents.

These documents are drafted by AI *before* large work begins, and *continuously updated* after each phase completes.

| Type | Prefix | Template | When to use |
|---|---|---|---|
| Multi-phase plan | `PXX` | `P00-planning-template.md` | Large changes requiring multiple F/R/B documents to complete |

Each PXX document contains:
- Background, overall objective, and scope of impact
- Description of each phase and **user-verifiable expected outcomes** (so the user can independently confirm whether each phase succeeded)
- Recommended F/R/B document types, and back-filled actual associated document numbers after execution
- Per-phase status (not started / in progress / completed)
- Execution instructions for the next AI agent, explaining how to read this document and advance to the next phase

---

### Category Zero-One — Queue (`documents/queue/`)

**Purpose:** Queue multiple ordered, automatically advanceable tasks so that AI completes them one by one using the DDD workflow, with each item using a new Codex / Claude Code session and an independent commit.

These documents are suited for scenarios where "I have 2 to 5 tasks I want AI to work through consecutively." Items can be independent or dependent; when dependent, `depends_on` and `unlock_condition` must be explicitly stated.

If the scope of subsequent items can only be known after a prior phase completes, or if the phase breakdown itself requires user review, use `ddd-plan`.

| Type | Prefix | Template | When to use |
|---|---|---|
| Work queue | `QXX` | `Q00-queue-template.md` | Multiple ordered, automatically advanceable features, bug fixes, or refactors |

Each QXX document contains:
- batch limit and blocked / completed notification settings; if email notification is needed, use `documents/ddd-email-notify.md` or QXX frontmatter to explicitly set `notify_email_from` (sending address) and `notify_email_to` (recipient address)
- `Queue Intake Review`: before execution, use `grill-me` centrally to clarify all items' requirements, design questions, dependencies, acceptance criteria, and stop conditions
- Per-item type (FXX/RXX/BXX), assigned agent, dependencies, unlock conditions, status, associated documents, and commit hash
- Per-item requirements, user-actionable acceptance criteria, and stop conditions
- `Agent Communication Ledger`: append-only record of task assignments, questions, answers, decisions, test evidence, and handoff summaries between the orchestrator, Codex, Claude Code, and the user; the QXX main document retains only the index, summaries, and active entries, with long logs archived to `documents/queue/logs/`
- Reason for blocked state and options requiring user decision

During execution, the orchestrator reads the QXX, first confirms that `Queue Intake Review` is complete, `ready_for_execution: true`, and that every item to be executed has `clarification_status: clarified`. It then starts a new worker session for each pending item. Workers handle only a single item and must fully execute `ddd-start → ddd-doc → ddd-tdd`. After completion, run tests, update documents, git commit, then stop.

Non-interactive sub-sessions do not engage in real-time conversation. When user judgment is needed, the worker sets the item to blocked, writes `questions` / `need_user_decision`, and appends a ledger entry; the orchestrator notifies the user and stops the queue. If `notify_email_from` and `notify_email_to` are set and a mailing tool is available in the current environment, the orchestrator sends a blocked notification; if settings are missing, the sender cannot be verified, or sending fails, report the blocked status in the current conversation and do not continue subsequent items. The `/ddd-tdd` called internally by queue workers does not send per-item completion emails; only when the entire batch queue is fully complete does the orchestrator send one completed notification. When the user replies, the orchestrator also appends an `answer` entry so subsequent agents can read the necessary communication records from the QXX. To control token usage, subsequent workers by default only read the Queue Intake Review, the designated item, dependent item handoffs, the Log Index, and relevant active entries; they do not read the full archive unless blocked or the context is contradictory.

---

### Category One — Implementation Documents (`documents/implements/`)

**Purpose:** Record and clarify the scope of each unit of work delivered to AI for implementation.

These documents are created *before* coding begins and updated *after* implementation is complete.

| Type | Prefix | Template | When to use |
|---|---|---|---|
| Feature spec | `FXX` | `F00-功能需求書模板.md` | New features |
| Refactor spec | `RXX` | `R00-重構任務模板.md` | Structural improvements that do not change behavior |
| Bug fix spec | `BXX` | `B00-Bug修正模板.md` | Reproduce and fix defects |

Each document contains:
- User story and feature background
- Acceptance criteria in Given/When/Then format
- Test scenario table (ID, Given, When, Then, priority)
- Implementation notes and constraints
- **Implementation record** section filled in by AI after coding (status, test evidence, changed files, assumptions, gaps)

---

### Category Two — Module Documents (`documents/modules/`)

**Purpose:** Reflect the current state of the codebase at a high level, so new engineers can quickly understand the system without reading every file.

Key characteristics:
- **High-level content only.** Responsibilities, boundaries, data flow, architecture — not line-by-line code details.
- **Always in sync with code.** When code and module documents are inconsistent, code takes precedence.
- **Engineer-facing.** Helps new members quickly locate a module's purpose, location, dependencies, and known limitations.
- **Does not describe unimplemented behavior.**

---

## Six DDD Skills

### `ddd-start` (new)

**Purpose:** Unified entry point. Detect requirement type and route to the correct downstream skill, ensuring no vague requirement goes directly into document writing.

**When to trigger:** At the start of any new task.

**Behavior:**
1. Read the project's `CONTEXT.md` (if it exists) to load domain language
2. Detect requirement type:
   - **Vague idea** (lacks concrete behavior description) → Run `grill-me`, then hand off to `ddd-doc` after convergence
   - **New feature involving existing architecture** (modifies existing modules) → Run `grill-with-docs`, hand off to `ddd-doc (FXX)`
   - **New feature with clear spec** → Hand off directly to `ddd-doc (FXX)`
   - **Refactor task** → Run `zoom-out` to confirm scope, hand off to `ddd-doc (RXX)`
   - **Bug report** → Hand off directly to `ddd-doc (BXX)`
   - **Long-running work queue** (2–5 ordered, automatically advanceable items) → Hand off to `ddd-queue`
   - **Large multi-phase change** (subsequent scope depends on earlier results, requires upfront planning or user review of phase breakdown) → Hand off to `ddd-plan`
3. When handing off, pass: requirement type, excavated context, suggested document prefix and number

---

### `ddd-plan` (new)

**Purpose:** Planning entry point for large changes. Break requirements that cannot be completed in one pass into ordered execution phases, define user-actionable outcomes for each phase, draft a PXX plan document for user review and approval, then hand off phase by phase to `ddd-doc` for execution.

**When to trigger:** When requirements span multiple modules, the phase breakdown requires user review, or subsequent phase scope can only be defined after a prior phase completes. If dependent items can be clearly listed upfront and automatically advanced, use `ddd-queue`.

**Behavior:**
1. Read `CONTEXT.md` to load domain language
2. If requirements are vague, run `grill-me` to deep-dive first, then return to plan
3. Assess scope of impact, identify module dependencies and the combination of change types
4. Break into phases following the principle: "each phase independently deliverable, corresponding to one F/R/B document, with an observable outcome"
5. Design **user-actionable confirmation steps** for each phase (not technical metrics)
6. Draft the PXX plan document, annotating each phase's suggested document type (FXX / RXX / BXX)
7. After user review and approval, PXX becomes the top-level basis for all subsequent F/R/B documents

**Phase-by-phase execution mode (after PXX approval):**

Each phase is handed to AI by the user; AI reads the current state and advances:
1. Read the PXX, find the earliest not-started phase
2. Call `ddd-doc` to draft the corresponding FXX / RXX / BXX (one at a time)
3. After user reviews and approves the F/R/B, call `ddd-tdd` to implement
4. After acceptance criteria pass, back-fill the PXX with the associated document number and update phase status
5. Repeat until all phases are complete, then change PXX `status` to `completed`

---

### `ddd-queue` (new)

**Purpose:** Long-running work queue entry point. Create or execute QXX queue documents, allowing 2 to 5 ordered, automatically advanceable features, bug fixes, or refactors to be completed one by one.

**When to trigger:** The user lists multiple tasks and wants AI to complete them one after another — each feature in a new session, with a git commit after each feature completes — or mentions queue / batch auto-execution / reducing interruptions.

**Behavior:**
1. Read `CONTEXT.md` and organize items using the project's domain language
2. Create `documents/queue/QXX-*.md`, or read an existing QXX
3. Use centralized `grill-me` first to clarify all items' requirements, design questions, dependencies, acceptance criteria, and stop conditions
4. Update `Queue Intake Review`, marking executable items as `clarification_status: clarified`
5. Verify that each item has clear requirements, acceptance criteria, and stop conditions
6. If there are dependencies between items, write `depends_on` and `unlock_condition`
7. If the scope of subsequent items can only be known after a prior phase completes, use `ddd-plan` instead
8. During execution, the orchestrator starts a new Codex / Claude Code worker session
9. Workers handle only the assigned item, executing `ddd-start`, `ddd-doc`, `ddd-tdd`, updating the queue, and git committing in sequence
10. All cross-agent messages are appended to the `Agent Communication Ledger`; summaries remain in the QXX, full content is archived to `documents/queue/logs/`
11. Stop after completing the batch limit or encountering a blocked item
12. When blocked, send an email notification to the user using `notify_email_from` / `notify_email_to`; if no mailing settings or tools are available, report the blocked status in the current conversation and keep subsequent items as pending
13. Send a completed notification when the entire batch queue finishes; do not send a completed notification when the batch limit is reached but pending items remain

---

### `ddd-email-notify` (new)

**Purpose:** Operational notification skill for the DDD workflow. Responsible for displaying current mailing info, configuring the project or QXX sender and recipient inbox, and handling email sending and ledger recording when ddd-tdd completes, a queue is blocked, or a queue completes.

**When to trigger:** User directly invokes `/ddd-email-notify` to view current mailing info, requests notification inbox configuration, or `ddd-tdd` / `ddd-queue` needs to send an email notification to the user.

**Behavior:**
1. Read `notify_email_from` and `notify_email_to` from `documents/ddd-email-notify.md` or QXX frontmatter
2. When triggered directly, display the current From, To, mailing tool status, and the timing for ddd-tdd completed / queue blocked / queue completed emails
3. If an older document only has `notify_email`, treat it as `notify_email_to` and migrate it on the next update
4. Verify that both email fields exist, and do not write passwords, tokens, SMTP keys, or app passwords to the QXX
5. Confirm that a mailing tool is available in the current environment, and that the sender matches the authorized account
6. Send completed or blocked notifications; if email cannot be sent, report it in the current conversation instead
7. Append the result of email success, failure, or fallback to the `Agent Communication Ledger`

---

### `ddd-doc` (enhanced doc-maintain)

**Purpose:** Create and maintain two categories of documents. Adds three plug-in points on top of the original `doc-maintain`.

**In the development cycle — before coding:**

1. Read `CONTEXT.md` to ensure document terminology is consistent with domain language
2. Assess requirement clarity — if vague, suggest running `grill-me` or `grill-with-docs` first
3. If the description is informal, use `to-prd` to convert it to a structured draft first, then fill in the FXX template
4. Find the next available FXX/RXX/BXX number and read the corresponding template
5. Check `documents/modules/` and the codebase for relevant context
6. **RXX document only:** Run `zoom-out` before locking in scope to confirm refactor boundaries are neither too narrow nor too wide
7. Draft a complete implementation document for user review

**In the development cycle — after coding:**

1. Update the FXX/RXX/BXX implementation record with the final results
2. Audit affected module documents against the new implementation
3. Update or flag outdated module documents
4. Ask: "Was any architecture technical debt discovered during implementation?" If so, suggest running `improve-codebase-architecture`, whose output can serve as a candidate draft for a new RXX document

---

### `ddd-tdd` (enhanced tdd-development)

**Purpose:** All production code changes, driven by an approved implementation document. Adds three plug-in points on top of the original `tdd-development`.

**Process (non-negotiable):**

1. Read `CONTEXT.md` to ensure test naming and behavior descriptions use the correct domain vocabulary
2. Read and understand the FXX/RXX/BXX document — this is the sole authoritative source of requirements
3. Extract acceptance criteria and test cases from the document
4. Write the minimal failing test (red phase) — must confirm failure before proceeding
5. **If a test fails unexpectedly (not a normal red light for "feature not yet implemented"):** Enter the `diagnose` structured debugging loop, and write assumptions, root cause, and fix strategy back to the "Implementation Record" in the FXX document
6. Implement the minimal production code needed to pass the test (green phase)
7. Verify that every acceptance criterion has test coverage
8. Write the implementation record back to the document
9. If this is not a queue worker, send the `/ddd-tdd` completed notification per `ddd-email-notify`; if this is a queue worker, suppress the per-item completion email
10. Ask: "Were any architecture gaps exposed during implementation?" If so, suggest `improve-codebase-architecture`
11. If this is a long-running implementation cycle, suggest `handoff` to preserve DDD state

Do not start from vague chat instructions. If no implementation document exists, stop and require one.

---

## On-demand Pocock Skills (Non-mandatory steps)

These skills can be manually triggered at any node in the DDD cycle and are not part of the mandatory flow:

| Skill | Role | Typical trigger |
|---|---|---|
| `caveman` | Compress communication, reduce token consumption | When repeatedly confirming wording during document review |
| `prototype` | One-off exploration to validate an approach | Highly uncertain features, before submitting FXX |
| `handoff` | Preserve DDD state across context resets | Long implementation cycles |
| `git-guardrails-claude-code` | Block destructive git commands | Safety infrastructure during ddd-tdd |
| `ddd-debug-trace` | Persist hard-bug investigation history in `documents/bugs/` and converge across sessions | A bug repeatedly resists root-cause analysis or requires multiple experiments |

### `ddd-debug-trace` (new)

**Purpose:** Autonomously investigate and fix complex bugs. In addition to the structured `diagnose` loop, it persists eliminated areas, confirmed facts, suspects, hypotheses, and experiment results in `documents/bugs/BUG-XX.md`, with a status snapshot at the top. Each new session resumes from the latest converged state.

**When to use:** The bug cannot be resolved in one pass, requires multiple experiments, or may need another session or agent. Use `diagnose` for an unexpected test failure within one session; use `ddd-doc (BXX) → ddd-tdd` for a straightforward reproducible defect whose fix location is known.

**Behavior:**
1. Scan `documents/bugs/`; resume from an existing status snapshot, or use `grill-me` in bug mode to establish precise symptoms, reproduction steps, and measurable completion criteria before creating a trace
2. Start from a clean working tree and iterate through declare experiment → change → measure → decide, writing every completed step back immediately
3. Commit partial improvements as new baselines; restore regressions or no-op changes and record the investigated area as eliminated
4. Treat eliminated areas and confirmed facts as monotonic knowledge unless contradictory evidence appears
5. When all metrics pass, record the solution and regression tests, set status to `resolved`, and hand off to `ddd-doc` to create the final BXX

---

## Key Design Principles

- **AI drafts, the user approves.** The document creation step is a collaborative checkpoint, not a formality.
- **No approved document, no code.** `ddd-tdd` treats the document as its sole prompt.
- **No test evidence, no completion.** A feature is not done unless tests have been run and passed.
- **Plan large changes before executing.** `ddd-plan` ensures multi-phase work has a clear map, with user-actionable acceptance steps for each phase rather than a vague "done" state.
- **Use a queue for automatically advanceable work.** `ddd-queue` is suited for 2 to 5 ordered items; they can be independent or dependent, but dependencies must be explicitly verifiable. Each item gets a new session, an independent commit upon completion, and stops when blocked.
- **Execute one phase at a time.** After PXX approval, draft F/R/B documents and implement phase by phase, avoiding early assumptions contaminating later design.
- **Debugging is structured, not ad hoc.** Every unexpected test failure is processed through `diagnose`, with results written to the FXX record.
- **Architecture observations are not lost.** Technical debt surfaced after implementation is converted into RXX candidate documents via `improve-codebase-architecture`.
- **Module documents reflect reality.** Outdated module documents are treated as defects by `ddd-doc`.
- **Pocock skills are plug-in points, not replacements.** `ddd-doc` and `ddd-tdd` remain the core execution engines; Pocock skills provide structured recovery paths at failure nodes.
