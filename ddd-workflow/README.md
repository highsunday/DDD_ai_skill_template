# DDD × Pocock Enhanced Workflow

Do not add the entire `ddd-workflow/` folder to the AI context. Load only the triggered `skills/*/SKILL.md`, and let the skill read templates or reference documents within the same folder as needed, to avoid consuming the entire token budget at once.

## Quick Start

1. Install or register `ddd-workflow/skills/` as triggerable skills
2. When initializing a project, trigger only `/ddd-create-folder`
3. `/ddd-create-folder` copies the F00 / R00 / B00 / P00 / Q00 / G00 templates from `skills/ddd-create-folder/templates/`
4. Fill in `CONTEXT.md` at the project root to define your project's domain terminology
5. Run `/ddd-start` to begin any new work; use `/ddd-queue` if you want the AI to process multiple ordered tasks continuously

## Development Loop

```
User submits a requirement
      ↓
/ddd-start   Detects type → routes to the correct entry point
      ↓
/ddd-doc     Drafts FXX / RXX / BXX documents
      ↓
User reviews and approves the document
      ↓
/ddd-tdd     Red → Green → Acceptance → Update document
```

Long-running work queues do not necessarily go through PXX:

```
User lists 2–5 ordered tasks
      ↓
/ddd-queue   Creates a QXX queue document
      ↓
Centralized /grill-me clarifies all items → updates Queue Intake Review
      ↓
Orchestrator opens a new Codex / Claude Code session for each item
      ↓
Worker runs ddd-start → ddd-doc → ddd-tdd → git commit → updates queue
      ↓
Agent Communication Ledger records dispatch / questions / answers / tests / handoffs
Long stdout / old history archived to documents/queue/logs/
      ↓
Stops after completing the batch limit or when blocked
```

DDD notification settings are stored in `documents/ddd-email-notify.md`; QXX can also override them via frontmatter. The email fields are `notify_email_from` (the sending address, which must be an address authorized to send in the current environment) and `notify_email_to` (the destination address). `/ddd-tdd` completing on its own sends a completed notification; `/ddd-queue` sends notifications when blocked and when the entire batch finishes; `/ddd-tdd` called by a queue worker does not send a per-item completion email. QXX does not store email passwords, tokens, or SMTP keys; when the email tool is unavailable, the orchestrator falls back to reporting in the current conversation.

## DDD Skills (Main Workflow)

| Skill | Purpose |
|---|---|
| `/ddd-create-folder` | **New project initialization**: creates the documents/ folder and F00 / R00 / B00 / P00 / Q00 / BUG00 / G00 templates |
| `/ddd-start` | Entry point for any new work; detects type and routes |
| `/ddd-doc` | Creates and maintains FXX / RXX / BXX documents and module documents |
| `/ddd-tdd` | Document-driven TDD implementation with integrated structured debugging |
| `/ddd-debug-trace` | Persists hard-bug investigation history across experiments and sessions, then hands the resolved root cause to a BXX |
| `/ddd-plan` | PXX multi-phase planning for large changes; use when the scope of later phases depends on completing earlier phases first |
| `/ddd-queue` | Continuously executes multiple ordered tasks; centralized grill-me clarifies all items first, then each item runs in a new session with its own commit; the QXX retains a compact ledger and archive index |
| `/ddd-email-notify` | Displays current email info, configures and sends DDD operation notifications; manages `notify_email_from` / `notify_email_to`, and explains when tdd completed, queue blocked, and queue completed notifications are sent |
| `/ddd-export-example` | Distills a successfully completed feature from this project into a standalone, verified green-light minimal example, exported to `reference-examples/export/` for other projects to reference |
| `/ddd-import-example` | References examples exported by other projects under `reference-examples/import/`, generates an FXX draft that cites the example (including acceptance criteria translated from the smoke test), and hands it to `/ddd-tdd` for implementation |

## Pocock Skills (On-Demand)

These skills are automatically suggested by `/ddd-start`, `/ddd-doc`, and `/ddd-tdd` at appropriate moments, and can also be triggered manually:

| Skill | Purpose | Typical trigger |
|---|---|---|
| `/grill-me` | Uncover ambiguous requirements | When an idea is not yet concrete |
| `/grill-with-docs` | Validate a plan against existing architecture | When a requirement involves existing modules |
| `/zoom-out` | Draw a module map for a high-level view | When confirming refactor scope or entering unfamiliar code |
| `/to-prd` | Convert informal descriptions to a structured draft | When there is a rough idea but no specification |
| `/diagnose` | Structured debugging loop | When a test fails unexpectedly |
| `/improve-codebase-architecture` | Surface architectural technical debt | After implementation, before planning a refactor |
| `/handoff` | Compact session state for the next conversation to continue | During long implementation cycles |
| `/caveman` | Compress communication to reduce tokens | When iterating on document wording repeatedly |
| `/prototype` | One-off exploratory validation of a solution | When a feature design has high uncertainty |
| `/git-guardrails` | Block destructive git commands | When setting up a project for the first time |

## Folder Structure

```
ddd-workflow/
├── README.md                      ← This file
├── CONTEXT.md                     ← Fill in your project's domain terminology (required)
├── WORKFLOW.md                    ← Full workflow documentation
└── skills/
    ├── ddd-start/   ← Entry routing
    ├── ddd-doc/     ← Document management
    ├── ddd-tdd/     ← TDD implementation
    ├── ddd-debug-trace/ ← Persistent hard-bug investigation
    ├── ddd-plan/    ← Large change planning
    ├── ddd-queue/   ← Long-running work queue
    ├── ddd-email-notify/ ← DDD tdd / queue notification email configuration, status display, and sending
    ├── ddd-export-example/ ← Export minimal runnable examples to reference-examples/export/
    │   └── templates/ ← MANIFEST templates
    ├── ddd-import-example/ ← Generate FXX from examples under reference-examples/import/
    ├── ddd-create-folder/
    │   └── templates/ ← Sole template source for F00 / R00 / B00 / P00 / Q00 / BUG00 / G00
    ├── grill-me/
    ├── grill-with-docs/
    ├── zoom-out/
    ├── to-prd/
    ├── diagnose/
    ├── improve-codebase-architecture/
    ├── handoff/
    ├── caveman/
    ├── prototype/
    └── git-guardrails/
```

## Where to Place Your Project Documents

Place implementation documents and module documents under your **project root**:

```
your-project/
├── documents/
│   ├── implements/   ← FXX / RXX / BXX work documents
│   ├── planning/     ← PXX multi-phase planning documents
│   ├── queue/        ← QXX long-running work queues
│   │   └── logs/     ← queue long logs / archives
│   └── modules/      ← High-level module documents
└── CONTEXT.md        ← Copy from ddd-workflow/CONTEXT.md and fill in
```

The sole template source is `ddd-workflow/skills/ddd-create-folder/templates/`. Do not maintain a second copy of templates in `ddd-workflow/documents/implements/`.
