---
name: ddd-create-folder
description: Creates the folder structure and document templates required for the DDD workflow in a new project. Sets up documents/ddd-email-notify.md, documents/implements/ (with F00 / R00 / B00 templates), documents/planning/ (with P00 template), documents/queue/ (with Q00 long-running work queue template and logs/ archive folder), documents/bugs/ (with BUG00 hard-bug investigation template), documents/modules/, documents/guides/ (with G00 template), and reference-examples/ (with export/ and import/ subfolders for storing cross-project examples used by ddd-export-example / ddd-import-example). Use when initializing the DDD workflow in a new project.
---

# DDD Create Folder

Creates the complete folder structure and initial template files required for the DDD workflow in the current project.

## Step 1 — Confirm creation location

Create the following structure in the current working directory (project root):

```
documents/
├── ddd-email-notify.md ← project-level email notification settings
├── implements/     ← storage for FXX / RXX / BXX work documents
│   ├── F00-feature-template.md
│   ├── R00-refactor-template.md
│   └── B00-bugfix-template.md
├── planning/       ← storage for PXX multi-phase planning documents
│   └── P00-planning-template.md
├── queue/          ← storage for QXX long-running work queues
│   ├── Q00-queue-template.md
│   └── logs/       ← storage for queue long logs / archives
├── bugs/           ← storage for BUG-XX hard-bug investigation traces
│   └── BUG00-trace-template.md
├── modules/        ← storage for module high-level documents (initially empty)
└── guides/         ← storage for GXX operation guides (running tests, building, etc.)
    └── G00-guide-template.md

reference-examples/  ← minimal runnable cross-project examples (top-level, independent of documents/)
├── export/          ← bundles exported from this project using ddd-export-example
└── import/          ← bundles copied in from other projects for ddd-import-example to reference
```

If the `documents/` folder already exists, notify the user and ask whether to continue (to avoid overwriting existing content).

## Step 2 — Create folders

Run the following commands:

```bash
mkdir -p documents/implements
mkdir -p documents/planning
mkdir -p documents/queue
mkdir -p documents/queue/logs
mkdir -p documents/bugs
mkdir -p documents/modules
mkdir -p documents/guides
mkdir -p reference-examples/export
mkdir -p reference-examples/import
```

## Step 3 — Write template files

Read the templates from this skill's folder (`skills/ddd-create-folder/templates/`) and write their contents unchanged into the project:

- `templates/F00-feature-template.md` → `documents/implements/F00-feature-template.md`
- `templates/DDD-email-notify-template.md` → `documents/ddd-email-notify.md`
- `templates/R00-refactor-template.md`  → `documents/implements/R00-refactor-template.md`
- `templates/B00-bugfix-template.md`   → `documents/implements/B00-bugfix-template.md`
- `templates/P00-planning-template.md` → `documents/planning/P00-planning-template.md`
- `templates/Q00-queue-template.md`    → `documents/queue/Q00-queue-template.md`
- `templates/BUG00-trace-template.md`  → `documents/bugs/BUG00-trace-template.md`
- `templates/G00-guide-template.md`    → `documents/guides/G00-guide-template.md`

If a target file already exists, **do not overwrite** it, and notify the user which files were skipped.

## Step 4 — Create CONTEXT.md (optional)

Ask the user: "Would you like to create a `CONTEXT.md` domain language template in the project root?"

If they agree, create a `CONTEXT.md` in the project root with the following content:

```markdown
# CONTEXT.md — Project Domain Language

> This is the project's domain vocabulary. All AI skills read this file before writing documents, tests, or code.
> Fill in your project's terminology and remove any examples that do not apply.

## Language (term definitions)

**[Term 1]**:
[Definition]
_Avoid using:_ [synonyms]

## Relationships

## Architecture Boundaries

## Flagged Ambiguities
```

If a `CONTEXT.md` already exists in the root, do not overwrite it; notify the user that it was skipped.

## Step 5 — Report results

Output a creation summary upon completion:

```
DDD folder initialization complete:

Created:
  ✓ documents/implements/
  ✓ documents/planning/
  ✓ documents/queue/
  ✓ documents/queue/logs/
  ✓ documents/bugs/
  ✓ documents/modules/
  ✓ documents/guides/
  ✓ reference-examples/export/
  ✓ reference-examples/import/
  ✓ documents/ddd-email-notify.md
  ✓ documents/implements/F00-feature-template.md
  ✓ documents/implements/R00-refactor-template.md
  ✓ documents/implements/B00-bugfix-template.md
  ✓ documents/planning/P00-planning-template.md
  ✓ documents/queue/Q00-queue-template.md
  ✓ documents/bugs/BUG00-trace-template.md
  ✓ documents/guides/G00-guide-template.md
  [✓ CONTEXT.md (if the user chose to create it)]

Skipped (already exists):
  [list skipped files, omit this section if none]

Next steps:
  1. Fill in CONTEXT.md to define your project's domain terminology
  2. Fill in notify_email_from / notify_email_to in documents/ddd-email-notify.md (if you need completion or blocked notifications)
  3. Run /ddd-start to begin your first task
```
