# DDD x Pocock — AI-Assisted Development Workflow

**DDD (Document-Driven Development)** means: write the document first, then write the code.

**AI drafts the document → you review and approve → AI implements from the document**

A "document" is not an after-the-fact explanation — it is a requirements contract that must exist before coding begins. AI does not write code directly from chat messages; it only writes code from documents you have approved.

---

## Why This Workflow

Unstructured AI development typically runs into four problems:

1. **Requirement drift** — You say "add a login," AI finishes it, and you realize it's not what you had in mind — but you can't articulate what's wrong because nothing was ever written down.
2. **No definition of "done"** — AI says it's finished, but you have no standard to verify against.
3. **Guessing when tests fail** — AI speculatively edits code, making things worse without finding the root cause.
4. **Technical debt disappears in conversation** — A design problem surfaces, but there's nowhere to record it, so the next time it comes up you start from scratch.

DDD has a corresponding design at every stage of the process to intercept each of these four problems.

---

## Three Rules That Cannot Be Broken

| Rule | Meaning |
|---|---|
| No approved document, no code | `ddd-tdd` stops without a document, forcing you to think it through first |
| No passing tests, not done | "Should be fine" does not count — test evidence is required before closing |
| Unexpected test failure requires structured debugging | Use `diagnose` to find the root cause — do not make speculative changes |

---

## Roles

**You:** provide requirements, review documents, approve, confirm acceptance results
**AI:** surface requirements, draft documents, write code, update records

The core principle is one thing: **AI drafts, but you approve**. That approval act is the quality gate for the entire workflow.

---

## Core Workflow

```
You describe the requirement
    ↓
/ddd-start   Detect requirement type, surface ambiguities if needed
    ↓
/ddd-doc     AI drafts the document (FXX / RXX / BXX)
    ↓
You review and approve
    ↓
/ddd-tdd     Red → Green → Acceptance
    ↓
/ddd-doc     Update implementation record, sync module documents
```

---

## How to Choose a Workflow

```
What is the requirement?
│
├─ Not clear yet           → /ddd-start → grill-me to surface it first
├─ Modifying existing module → /ddd-start → grill-with-docs to validate architecture
├─ Brand new feature       → /ddd-start → ddd-doc FXX
├─ Refactor                → /ddd-start → zoom-out → ddd-doc RXX
├─ Bug (root cause known)  → /ddd-start → ddd-doc BXX
├─ Bug (root cause unknown)→ /ddd-start → ddd-debug-trace (persist investigation history and converge)
├─ Multiple items at once  → /ddd-queue (each item has clear requirements and acceptance criteria)
└─ Large-scale change      → /ddd-plan (later details depend on earlier results)
```

**Queue or plan?**
- You can list all items' requirements and acceptance criteria right now → **queue**
- What comes next depends on what the earlier phases produce → **plan**

---

## Command Reference

### `/ddd-create-folder` — Initialize the project

Creates the DDD folder structure and initial templates in a new project. **Use this the first time you introduce DDD.**

```
documents/
├── ddd-email-notify.md         ← notification email settings
├── implements/                 ← FXX / RXX / BXX
├── planning/                   ← PXX multi-phase plans
├── queue/                      ← QXX work queues
│   └── logs/
├── bugs/                       ← BUG-XX hard-bug investigation traces
├── modules/                    ← high-level module documents
└── guides/                     ← GXX operation guides
CONTEXT.md                      ← domain glossary (optional)
```

After initialization: fill in `CONTEXT.md` to define project terminology → configure email notification settings → `/ddd-start` to begin the first task.

---

### `/ddd-start` — Entry point for all new work

Detects requirement type and routes to the correct next step, ensuring no ambiguous requirement goes directly into document drafting. **Start every new task here.**

| Requirement type | Routes to |
|---|---|
| Vague idea | `grill-me` → `ddd-doc FXX` |
| New feature touching existing architecture | `grill-with-docs` → `ddd-doc FXX` |
| Well-specified new feature | `ddd-doc FXX` |
| Refactor task | `zoom-out` → `ddd-doc RXX` |
| Bug report (root cause known, clean fix) | `ddd-doc BXX` |
| Bug report (root cause unknown, hard bug) | `ddd-debug-trace` |
| 2–5 items to complete in one go | `ddd-queue` |
| Multi-phase large-scale change | `ddd-plan` |

Keywords that trigger `ddd-plan` directly: "help me plan," "let's plan this out," "phase-by-phase plan," "PXX"

---

### `/ddd-doc` — Draft and maintain documents

**Before coding:** read `CONTEXT.md` → confirm requirement clarity → draft the document (including acceptance criteria and test scenarios) → hand to you for review.

**After coding:** update the implementation record → audit module documents → ask whether any architectural technical debt was identified.

Each document contains: user story, Given/When/Then acceptance criteria, test scenario table, implementation notes, and an **implementation record** (filled in after completion).

---

### `/ddd-tdd` — Implement from the document

**Prerequisite: an approved FXX / RXX / BXX must exist — otherwise stop.**

1. Read the document and extract acceptance criteria
2. **Red:** write the minimal failing test and confirm it fails
3. If a test fails unexpectedly → enter `diagnose` for structured debugging, write the root cause back into the document
4. **Green:** implement the minimal production code
5. **Acceptance:** confirm every acceptance criterion has test coverage
6. Write the implementation record back into the document

---

### `/ddd-debug-trace` — Investigate hard bugs autonomously

Use this for bugs whose root cause is unknown and requires multiple experiments to isolate. It persists the investigation in `documents/bugs/BUG-XX.md`, so each AI session resumes from the latest converged state instead of repeating earlier work.

**Difference from BXX:** BXX specifies a fix when the change location is already understood. `ddd-debug-trace` records the investigation while the root cause is still unknown.

**How it works:**
1. Scan `documents/bugs/`; resume an existing trace from its status snapshot, or use `grill-me` to establish precise symptoms, reproduction steps, and measurable completion criteria before creating a BUG-XX document
2. Start from a clean working tree and iterate: **declare experiment → change → measure → decide**, writing each result back immediately
3. Commit partial improvements as new baselines; restore regressions or no-op changes and record those areas as eliminated
4. Treat confirmed facts and eliminated areas as monotonic knowledge; do not reinvestigate them without contradictory evidence
5. When resolved, hand off to `ddd-doc` to create a BXX containing the root cause, fix, and regression tests, with links in both directions

It can also be entered from `ddd-tdd` when an apparently simple BXX stalls and `diagnose` cannot identify the root cause.

---

### `/ddd-plan` — Plan large-scale changes

Breaks multi-phase requirements into an ordered execution plan, drafts a PXX planning document, with each phase corresponding to one F/R/B document.

**When to use:** the requirement spans multiple modules, later details depend on earlier results, or the phase breakdown itself needs your review.

**After PXX is approved, execute phase by phase:**
```
You say "continue PXX" → AI finds the earliest unstarted phase → drafts F/R/B
→ you approve → ddd-tdd implements → backfill PXX status → repeat until done
```

**Example: E-commerce admin permission system overhaul**

You say: "I want to migrate the admin permission system from role-based to resource-based, refactor the old admin API at the same time, and finally build the review workflow UI. Help me plan this."

AI drafts `documents/planning/P01-permission-refactor.md`:

```
P01 — Admin Permission System Overhaul

Phase 1: Refactor old admin API (RXX)
  Your acceptance: log in with an existing account, all features work, no regressions
  Status: not started

Phase 2: Implement resource-based permission model (FXX)
  Your acceptance: can configure "who can do what action on which resource" in the admin panel
  Depends on: Phase 1 complete
  Status: not started

Phase 3: Review workflow UI (FXX)
  Your acceptance: can see the pending review list, click to approve or reject with immediate feedback
  Depends on: Phase 2 complete
  Status: not started
```

After you approve, one phase at a time:
```
"Please continue P01 and advance to the next phase."
→ AI drafts R01 → you approve → ddd-tdd implements → backfill P01
```

---

### `/ddd-queue` — Batch automated execution

Lets AI complete 2–5 ordered tasks one by one, each item in a new session, with an independent commit after each. **Use this when you want fewer interruptions and continuous AI execution.**

**Execution flow:**
1. Create the QXX document
2. Use `grill-me` in one focused session to clarify all items' requirements, acceptance criteria, and stop conditions
3. You confirm the QXX and allow execution
4. AI opens a new session per item, runs `ddd-start → ddd-doc → ddd-tdd`, and commits after completion
5. If blocked, stop immediately and notify you

**Example 1: Three independent bug fixes**

You say: "I have three bugs to fix, requirements are all clear — queue them up and run through them, email me if blocked."

AI creates `documents/queue/Q01-bug-fixes.md`:

```yaml
batch_limit: 3
notify_email_from: bot@myproject.com
notify_email_to: you@example.com

items:
  - id: Q01-01
    title: Fix duplicate submission on login page
    type: BXX
    clarification_status: clarified
    acceptance: Clicking the login button twice quickly sends only one request

  - id: Q01-02
    title: Fix incorrect amount display in order list
    type: BXX
    clarification_status: clarified
    acceptance: Amount shown in order list matches the order detail page

  - id: Q01-03
    title: Fix user avatar not refreshing after upload
    type: BXX
    clarification_status: clarified
    acceptance: Page updates immediately after avatar upload without requiring a reload
```

Execution order:
```
Q01-01: new session → ddd-doc BXX → ddd-tdd → git commit → stop
Q01-02: new session → ddd-doc BXX → ddd-tdd → git commit → stop
Q01-03: new session → ddd-doc BXX → ddd-tdd → git commit → stop
→ entire batch complete, send email notification
```

**Example 2: Queue with dependencies**

You say: "Build the user tagging feature first, then build the tag-based search filter. Notify me when done."

```yaml
items:
  - id: Q01-01
    title: User tagging feature
    type: FXX
    clarification_status: clarified
    acceptance: Can add/remove tags on a user; tags persist after page reload

  - id: Q01-02
    title: Filter user search by tag
    type: FXX
    depends_on: Q01-01
    unlock_condition: Q01-01 complete and tag API available
    clarification_status: clarified
    acceptance: Search page allows selecting a tag filter; results show only users with that tag
```

After Q01-01 completes and is committed, AI automatically unlocks Q01-02 and continues.

---

### `/ddd-email-notify` — Notification settings

View or configure email notifications for blocked or completed events.

| Event | Send email |
|---|---|
| `/ddd-tdd` single completion | Yes |
| Queue single item completion | No (handled by batch summary) |
| Queue entire batch complete | Yes |
| Queue blocked | Yes |

Configuration location: `documents/ddd-email-notify.md` (project level) or QXX frontmatter (per queue).

---

## Supporting Commands

| Command | Purpose | Typical use case |
|---|---|---|
| `/grill-me` | Surface the real requirement by asking questions one by one to clarify boundaries | Requirement is vague and acceptance criteria cannot be written |
| `/grill-with-docs` | Validate the plan against the existing domain model | New feature requires modifying an existing module |
| `/diagnose` | Structured debugging: reproduce → hypothesize → fix → regression test | Test fails unexpectedly and the cause is unknown |
| `/ddd-debug-trace` | Persist a hard-bug investigation across experiments and sessions | Root cause remains unknown after initial diagnosis |
| `/zoom-out` | Draw a module map to confirm refactor scope | Confirm impact boundaries before starting a refactor |
| `/handoff` | Compress the current conversation into a handoff document | Long implementation session that needs to switch to a new session |
| `/prototype` | Build a throwaway prototype to validate a design direction | Highly uncertain feature, before committing to an FXX |
| `/improve-codebase-architecture` | Find opportunities to deepen architecture, output RXX candidates | Technical debt discovered after implementation |
| `/caveman` | Switch to ultra-minimal communication mode to reduce tokens | Repeated wording clarifications during document review |
| `/git-guardrails` | Block dangerous git commands | Safety protection during implementation |

---

## Document Types

| Prefix | Type | Location | When to use |
|---|---|---|---|
| `FXX` | Feature spec | `documents/implements/` | New features |
| `RXX` | Refactor spec | `documents/implements/` | Structural improvements that do not change behavior |
| `BXX` | Bug fix spec | `documents/implements/` | Reproduce and fix a defect |
| `PXX` | Multi-phase plan | `documents/planning/` | Large-scale changes spanning multiple modules |
| `QXX` | Work queue | `documents/queue/` | 2–5 tasks that can be auto-advanced |
| `BUG-XX` | Bug investigation trace | `documents/bugs/` | Hard bugs requiring multiple experiments; produces a BXX after resolution |
| `GXX` | Operation guide | `documents/guides/` | Instructions for running tests, packaging builds, etc. |

Module documents (`documents/modules/`): high-level descriptions of an existing module's responsibilities, boundaries, dependencies, and known limitations. Updated in sync with code; when code and document conflict, the code takes precedence.

---

## Quick Reference by Scenario

| Scenario | Command |
|---|---|
| New feature | `/ddd-start` → describe requirement → approve FXX → `/ddd-tdd` |
| Vague requirement | `/ddd-start` → AI runs grill-me automatically → enter ddd-doc after converging |
| Bug discovered (root cause known) | `/ddd-start` → describe the symptom and reproduction steps → approve BXX → `/ddd-tdd` |
| Hard bug (root cause unknown) | `/ddd-start` → `/ddd-debug-trace` → investigate until resolved → create BXX |
| Refactor | `/ddd-start` → zoom-out to confirm scope → approve RXX → `/ddd-tdd` |
| Multiple items in batch | `/ddd-start` → ddd-queue → confirm QXX → auto-execute |
| Large multi-phase change | `/ddd-start` → say "help me plan" → approve PXX → execute phase by phase |
| Introducing DDD to a new project | `/ddd-create-folder` → fill in CONTEXT.md → `/ddd-start` |
