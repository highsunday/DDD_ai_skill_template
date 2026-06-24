---
name: ddd-doc
description: DDD enhanced documentation skill. Creates and maintains engineering documents under documents/, intercepting vague requirements before drafting, confirming scope during RXX, and surfacing architectural technical debt after coding. Use when drafting or updating FXX / RXX / BXX documents, or auditing module documents.
---

# DDD Doc

Manages engineering documents under `documents/`, integrating Pocock skills at three key hook points to improve document quality.

## Document Structure

```
documents/
├── implements/
│   ├── FXX-*   Feature requirement documents
│   ├── RXX-*   Refactor task documents
│   ├── BXX-*   Bug fix documents
│   └── Templates: F00 / R00 / B00
└── modules/
    └── High-level module documents (architecture, responsibilities, data flow)
```

## Core Goal

Keep documents and implementation in sync. When code and documents conflict, treat the current implementation as the source of truth, unless the user explicitly states that the document describes intended future behavior.

---

## Creating FXX / RXX / BXX Documents

### Prerequisites: Load Domain Language

Read `CONTEXT.md` (if it exists). All terminology in documents must align with it. If you find that terminology in the request conflicts with definitions in `CONTEXT.md`, raise it immediately: "Your glossary defines X as A, but you seem to mean B — please confirm which is correct?"

### Hook A — Before Drafting: Intercept Vague Requirements

Before drafting any document, assess requirement clarity:

**If the requirement is vague** (lacks specific behavior, cannot derive test cases):
- Inform the user: "This requirement is not specific enough. It is recommended to run `/grill-me` to dig deeper before drafting the document."
- Pause and wait for the user's decision.

**If the requirement involves existing architecture but the impact scope is unconfirmed**:
- Read the relevant module documents in `documents/modules/`
- Inform the user: "This requirement affects [module name]. It is recommended to run `/grill-with-docs` to validate boundaries against the existing architecture before drafting."
- Pause and wait for the user's decision.

**If the requirement description is informal** (natural language, no Given/When/Then structure):
- Proactively suggest: "I can first use `/to-prd` to convert the description into a structured draft, then fill in the document template — it will be faster. Would you like to do that?"

**If the requirement is clear**: proceed directly to the next step.

### Step 1 — Confirm Document Type and Number

1. Confirm the type (new feature FXX / refactor RXX / bug fix BXX)
2. List `documents/implements/` to find the next available number
3. Read the corresponding template (F00 / R00 / B00)
4. Read existing documents adjacent in numbering to understand established conventions

### Step 2 — Gather Context

- Search `documents/modules/` for relevant feature names, components, services, routes
- Read relevant module documents
- Use `rg` to search the codebase for related files and implementation terms
- Use module documents to understand high-level architecture, but verify important details against source code

### Hook B — During RXX Documents: Confirm Refactor Scope

**Execute only when creating an RXX document:**

Execute `zoom-out` — ask the AI to map all related modules and callers, described using the terminology from `CONTEXT.md`.

After confirmation, ask the user:
- "Does this module map cover all the parts that need to be changed?"
- "Are there any boundaries drawn too narrowly (missing coupling) or too broadly (exceeding the scope of this task)?"

Only continue drafting the RXX document after the scope is confirmed.

### Step 3 — Draft Document

Follow the template structure and ensure:

- **Acceptance criteria are testable.** Each criterion maps directly to a test case.
- **Includes specific test scenarios.** Use a table (ID, Given, When, Then, Priority).
- **Lists modules and files that may be affected.**
- **Clearly marks assumptions, open questions, and non-goals.**
- **Does not over-specify implementation details**, unless correctness or compatibility requires it.

### Step 4 — Pre-completion Check

- Confirm whether new requirements need new module documents created or existing ones updated
- Inform the user: "It is recommended to create/update the following module documents: [list]"

---

## After Implementation: Sync Documents

### Step 1 — Update Implementation Record

Reopen the relevant FXX/RXX/BXX document and fill in the "Implementation Record" section with:
- Final behavior
- Changed files / modules
- Added or updated test coverage
- Tests executed and results
- Assumptions and decisions (especially findings from the `diagnose` process)
- Known limitations or follow-up work

### Step 2 — Audit Module Documents

Compare against `documents/modules/`:
- If an existing module has significant changes, update its document
- If a new module, feature area, service, or architectural boundary was introduced, suggest creating a new module document
- If the user requests a full sync, execute updates directly; otherwise provide specific suggestions

### Hook C — After Coding: Surface Architectural Technical Debt

After updating the implementation record, ask the user:

> "Were any architectural issues discovered during implementation (e.g., overly tight module coupling, lack of appropriate test seams, unclear responsibility boundaries)?"

- If yes: suggest running `/improve-codebase-architecture`, explaining that its output can serve directly as a candidate draft for a new RXX document
- If no: the process ends normally

---

## Auditing Module Documents

1. Read the relevant documents in `documents/modules/`
2. Search the implementation for every significant claim (framework versions, routes, components, data sources, state flow, API behavior, tests, known limitations)
3. Categorize inconsistencies:
   - Outdated paths or file structures
   - Missing modules or behaviors
   - Code changed but document not updated
   - Document describes future intent as current implementation
   - Too many low-level details (prone to going stale)
4. Update module documents to reflect current code — keep them high-level, engineer-oriented, and include file paths as navigation anchors
5. Point out code areas that still lack module documentation

---

## Module Documentation Writing Standards

A good module document section covers:

- Purpose
- Current implementation status
- Key files
- Primary responsibilities
- Data / state flow
- External dependencies
- Important constraints
- Known limitations
- Testing notes
- Related FXX/RXX/BXX documents

Avoid:
- Copying large code snippets
- Documenting every prop or local variable
- Describing unimplemented behavior as existing
- Repeating details that are already obvious from the code
- Leaving outdated paths or old folder names

---

## Verification Habits

Prefer `rg` or `rg --files` before searching.

Before editing any document, you must first review:
- `documents/implements/`
- `documents/modules/`
- Relevant source code files
- `package.json` when framework / library versions matter
- Tests or test configuration when documenting how verification was done

When uncertain, explicitly state the uncertainty rather than fabricating missing implementation details.
