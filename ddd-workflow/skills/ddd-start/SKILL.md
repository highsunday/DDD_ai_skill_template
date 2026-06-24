---
name: ddd-start
description: Unified entry point for the DDD workflow. Detects requirement type and routes to the correct downstream skill, ensuring no vague requirements proceed directly to document writing. Use when a user raises new work (feature, refactor, bug, vague idea).
---

# DDD Start

Entry point for all new work. Execute before any document writing or code implementation begins.

## Step 1 — Load domain language

Look for `CONTEXT.md` in the root directory:

- If it exists: read it and remember the glossary and domain boundaries within. All terminology in subsequent conversations and documents must be consistent with it.
- If it does not exist: inform the user "This project is missing CONTEXT.md. It is recommended to build it incrementally during implementation to lock down domain vocabulary." Do not block the flow; continue to the next step.

## Step 2 — Detect requirement type

Read the user's original request and determine which of the following types it belongs to. If you cannot determine the type, ask the user directly: "Is this a new feature, refactor, bug fix, or still in the ideation stage?"

### Type A — Vague idea
**Characteristics:** Lacks specific behavior descriptions, has no clear acceptance criteria, uses words like "maybe", "roughly", "feels like".

**Route:**
1. Inform the user: "The requirement is not specific enough yet. Let's use `/grill-me` to dig deeper first."
2. Execute the `grill-me` drilling process: ask questions one by one to clarify target users, core behaviors, and boundary conditions.
3. Once the drilling converges, organize the conclusions from the conversation into a structured description and hand off to `ddd-doc`, specifying type FXX.

### Type B — New feature involving existing architecture
**Characteristics:** Explicitly mentions modifying existing modules, services, or APIs; or the new feature depends on existing domain concepts.

**Route:**
1. Inform the user: "This requirement involves existing architecture. Let's use `/grill-with-docs` to validate against the domain model first."
2. Execute the `grill-with-docs` process: compare against `CONTEXT.md`, `documents/modules/`, and the codebase to challenge contradictions in the plan.
3. After validation, hand off to `ddd-doc` specifying type FXX.

### Type C — Well-specified new feature
**Characteristics:** The requirement includes clear behavior descriptions; acceptance conditions can directly derive test cases.

**Route:** Hand off directly to `ddd-doc`, specifying type FXX.

### Type D — Refactor task
**Characteristics:** Clearly describes improving structure, eliminating technical debt, or optimizing readability, without changing external behavior.

**Route:**
1. Execute `zoom-out`: ask the AI to draw a map of the relevant modules to confirm the refactor scope.
2. Ask the user: "Does this module map cover all the parts that need to be changed?"
3. After confirmation, hand off to `ddd-doc`, specifying type RXX.

### Type E — Bug report
**Characteristics:** Describes "existing behavior not matching expectations", with clear error symptoms or reproduction steps.

Use one question to route it: **"Do we currently know where the fix belongs?"**

#### E1 — Root cause known, clean fix
**Characteristics:** The defect is reproducible and the likely fault location and correction are understood, such as a missing null check or an incorrect error-message string.

**Route:** Hand off directly to `ddd-doc`, specifying type BXX, then follow BXX → `ddd-tdd`.

#### E2 — Root cause unknown, hard bug
**Characteristics:** The root cause is unknown, multiple experiments are required, reproduction is intermittent, or the user says the issue has resisted prior investigation. Resolution may span sessions or agents.

**Route:**
1. Tell the user: "The root cause is still unclear. Switching to `/ddd-debug-trace` to persist the investigation history and converge incrementally."
2. Hand off to `ddd-debug-trace`: scan `documents/bugs/`; if no matching trace exists, use `grill-me` in bug mode to establish precise symptoms, reproduction steps, and measurable completion criteria, then create `documents/bugs/BUG-XX.md` and investigate autonomously.
3. After status reaches `resolved`, hand off to `ddd-doc` to create a BXX containing the root cause, fix, and regression tests.

If E1 versus E2 is unclear, start with E1. Escalate to `ddd-debug-trace` if `ddd-tdd` and `diagnose` cannot identify the root cause.

### Type F — Long-running work queue

**Characteristics:** The user lists 2 to 5 ordered features, bug fixes, or refactors they want the AI to complete one by one, to extend AI autonomous work time and reduce human interruptions; common keywords include "queue", "batch", "complete one by one", "commit per feature", "new session per feature", "Codex and Claude take turns", "let AI do them consecutively".

**Difference from ddd-plan:**
- If the user can already list specific items, and each item has requirements, acceptance criteria, and stop conditions; and dependencies can be described with `depends_on` and `unlock_condition` → use `ddd-queue`
- If the scope of later items can only be known after earlier phases are complete, or the phase breakdown itself requires human review → use `ddd-plan`

**Route:**
1. Hand off to `ddd-queue` to create or read `documents/queue/QXX-*.md`.
2. During the QXX creation phase, first run `grill-me` in a concentrated session to clarify all items' requirements, design questions, dependencies, acceptance criteria, and stop conditions in one go.
3. Only when the QXX has `intake_grill_status: completed`, `ready_for_execution: true`, and all pending items have `clarification_status: clarified` is execution permitted.
4. If the user requests immediate execution, explicitly state that the orchestrator will launch a new Codex / Claude Code session, one session per item.
5. If any item lacks verifiable behavior, require it to be filled in or mark that item as blocked; do not proceed directly to implementation.

### Type G — Large multi-phase change

**Do not default to this route.** When in doubt, first try describing the requirement in a single F/R/B document; if the user has already listed multiple items that can be automatically progressed, use Type F's `ddd-queue`.

**Explicit trigger (priority):** If the user says any of the following keywords, route directly without further judgment:
- "help me plan", "let's plan", "plan first", "make a plan", "write a plan"
- "phase-by-phase planning", "staged planning", "PXX phases"
- "PXX", "Planning", "planning doc"

**Automatic judgment (cautious):** Only auto-route if **all** of the following conditions are met simultaneously:
1. A complete F/R/B document cannot be written now (because later requirement details depend on earlier completion)
2. There are clear execution dependencies (B truly cannot begin before A is complete, not merely a matter of priority)
3. The nature of different parts is fundamentally different (e.g., the architecture must be refactored before new features can be built on top)

**Do not use this route when:**
- The requirement is complex but can be fully described in a single document → use Types A-E
- Multiple tasks can already be listed as a clear queue, and you simply want them completed automatically or with fewer interruptions → use Type F
- Multiple modules are involved but they belong to the same feature → use Type C or B
- It just "feels big" → try `ddd-doc` first; only plan if it does not fit in one document

**Route:**
1. If triggered by keyword: hand off directly to `ddd-plan` and inform the user "Got it, starting PXX planning."
2. If auto-judged: inform the user "The later details of this change depend on earlier results; it is recommended to use `/ddd-plan` to plan each phase first." Then hand off after confirmation.

## Step 3 — Hand off

**When handing off to `ddd-doc` (Types A-E1), explicitly state the following three points:**

1. **Document type:** FXX / RXX / BXX
2. **Collected context:** A summary of conclusions formed during the grill-me / grill-with-docs / zoom-out process
3. **CONTEXT.md status:** Loaded / Does not exist

Example:
> "Requirement drilling complete. Type: FXX. Core requirement: [summary]. CONTEXT.md loaded, key terms: [term list]. Please continue with `/ddd-doc`."

**When handing off to `ddd-debug-trace` (Type E2), explicitly state the following three points:**

1. **Symptom summary:** The observed failure and error fingerprint, including exact messages, stack traces, or anomalous values
2. **Known reproduction information and completion criteria:** Reproduction steps, frequency, and measurable success thresholds; `ddd-debug-trace` uses `grill-me` to fill gaps
3. **CONTEXT.md status:** Loaded / Does not exist

Example:
> "This is a hard bug with an unknown root cause; route to `/ddd-debug-trace`. Symptom: [summary]. Error fingerprint: [message]. Reproduction: [steps], frequency: intermittent. Completion criteria still need measurable thresholds. CONTEXT.md loaded. Create `documents/bugs/BUG-XX.md`, investigate autonomously, and hand off to `/ddd-doc` for a BXX after resolution."

**When handing off to `ddd-plan` (Type G), explicitly state the following three points:**

1. **Requirement description:** A summary of the user's original requirement
2. **Known impact scope:** Currently identifiable modules or functional areas
3. **CONTEXT.md status:** Loaded / Does not exist

Example:
> "This requirement spans multiple modules and is large in scale. Known impact scope: [module list]. CONTEXT.md loaded. Please continue with `/ddd-plan` to break down phases and draft a PXX planning document."

**When handing off to `ddd-queue` (Type F), explicitly state the following six points:**

1. **Queue goal:** The degree to which this batch of work should be autonomously completed by the AI
2. **Item list:** Each item's name, type (FXX/RXX/BXX), acceptance criteria, dependencies, and unlock conditions
3. **Execution settings:** batch limit, preferred agent, whether to send email on blocked / completed; if email is needed, provide `notify_email_from` (sending address) and `notify_email_to` (recipient address)
4. **Concentrated clarification status:** Completed / Need to run grill-me first
5. **Context strategy:** The QXX master document retains only index, summary, active entries, and handoff; long logs are archived to `documents/queue/logs/`
6. **CONTEXT.md status:** Loaded / Does not exist

Example:
> "This is a batch of ordered and automatically progressable work, suitable for `/ddd-queue`. There are 3 items in total: Q02 depends on Q01, Q03 depends on Q02. Default batch limit is 3; send email notification when blocked and when the entire batch is completed; the sending address is `notify_email_from`, the recipient inbox is `notify_email_to`. `/ddd-tdd` called internally by the queue worker does not send per-item completion emails; only the queue sends one email when the entire batch is complete. CONTEXT.md loaded. Please first create the QXX queue document and run `/grill-me` in a concentrated session to clarify all items; once QXX is ready, if the user requests immediate execution, act as orchestrator and launch a new Codex / Claude Code session for each item. QXX uses compact context: the master document retains summary, index, active entries, and handoff; long logs are archived."
