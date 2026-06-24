---
name: ddd-plan
description: DDD multi-phase planning skill. Breaks down large-scale changes that cannot be completed in one pass — where phase breakdown or subsequent scope requires human review — into ordered execution phases. Designs user-verifiable expected outcomes for each phase and drafts a PXX planning document for human review. Once approved, it serves as the parent reference for subsequent FXX / RXX / BXX documents. If the user can already list 2 to 5 specific work items that can be automatically progressed, use ddd-queue instead.
---

# DDD Plan

Entry point for planning large-scale changes. Responsible for breaking down vague or large requirements into independently executable phases, defining outcomes that humans can actually verify for each phase, then handing off to subsequent `ddd-doc` + `ddd-tdd` for phase-by-phase execution.

## Core rules

**No clear breakdown, no PXX.**

If the requirement itself is too vague to determine how many phases it needs or which modules it affects, run `grill-me` to dig deeper first, then come back to plan.

**PXX is not a design document — it is an execution map.**

The purpose of PXX is to let the handing-off AI know where things currently stand and what the next step should be. Each phase's "user verification method" must be an action a human can actually perform — not "tests pass" or "code merged."

---

## When to use ddd-plan

**Default: do not use PXX.** When in doubt, try `ddd-doc` first — if one document can fully describe everything, no planning document is needed.

### Should use ddd-plan

**Explicit trigger (user says a keyword):**
If the user explicitly says "help me plan," "plan this out," "phase-by-phase planning," "PXX," etc., go directly into planning without further evaluation.

If the user says "queue," "batch execution," "new session per feature," "commit per feature," "let AI run continuously," and has already listed specific items that can be automatically progressed, use `ddd-queue` — do not create a PXX.

**Automatic evaluation (all conditions must be met simultaneously):**
1. A complete F/R/B document cannot be written right now (later details depend on earlier results)
2. There are real planning dependencies (the scope or approach of B cannot be determined until A is finished)
3. The nature of different parts is fundamentally different (e.g., the underlying layer must be refactored before deciding how subsequent features land)

### Should not use ddd-plan

- Requirement is complex but can be fully described in one document → go directly to `ddd-doc`
- Changes span multiple modules but belong to the same feature → a single FXX is sufficient
- It just "feels large" → write an F document first; only plan if it doesn't fit on one page
- There is ordering or clear dependency, but all items can be clearly listed upfront and automatically progressed → use `ddd-queue`
- There is ordering but no dependency (A and B can both run independently, just do A first) → create two separate F documents, or use `ddd-queue` for batch processing

---

## Step 1 — Load domain language

Look for `CONTEXT.md` in the root directory:

- If it exists: read it and memorize the glossary. All subsequent document terminology must be consistent with it.
- If it does not exist: continue the process, use vocabulary from the user's description when drafting the PXX, and suggest adding `CONTEXT.md` afterward.

---

## Step 2 — Intercept vague requirements

Before entering the breakdown, assess the actionability of the requirement:

**If the requirement is extremely vague** (cannot identify change boundaries or which modules will be affected):
- Inform the user: "The scope of this requirement is not yet clear enough. Use `/grill-me` to dig deeper before planning."
- Pause and wait for the user to decide.

**If the requirement is broadly clear but details need confirmation:**
- Continue to Step 3 and proactively annotate uncertain points during the drafting phase for user review.

**If the requirement is already clear:** go directly to Step 3.

---

## Step 3 — Understand the full picture

Before breaking down phases, fully understand the overall change:

1. **Confirm the overall goal**: Describe in one sentence "after everything is done, what different experience will the user have."
2. **Inventory the impact scope**: List the modules or functional areas this change is expected to affect.
   - Read relevant module documents in `documents/modules/` (if it exists)
   - Use `rg` to search the codebase for relevant module names, services, routes, components
3. **Identify dependencies**: Which parts must be completed before the next part can begin?
4. **Confirm the mix of change types**: Are each of the parts a new feature (F), refactor (R), or bug fix (B)?

---

## Step 4 — Phase breakdown

### Breakdown principles

Each phase must satisfy:

- **Independently deliverable**: After this phase is complete, the system remains in a runnable, usable state.
- **Clear boundaries**: Does not cross more than two major functional areas.
- **Maps to one document**: Each phase should be fully describable by a single FXX / RXX / BXX document.
- **Has an observable outcome**: The user can perform a specific action at the end of this phase to confirm it is complete.

**Poor phase breakdown (avoid):**
- Phase 1 "complete backend" / Phase 2 "complete frontend" — cuts across technical layers rather than functional boundaries; the user cannot confirm any behavior at the end of Phase 1
- Too many phases (more than 6) — usually indicates the requirement itself should be reduced in scope

### Recommended decomposition order

When in doubt, arrange phases in the following priority order:

1. **Infrastructure / data model** (if needed) — refactor, ensuring subsequent features have a solid foundation
2. **Core functionality** (highest business value) — make the most important user behavior available first
3. **Extended functionality** — expand on the core foundation
4. **Cleanup and fixes** — bugs or technical debt discovered during integration

### Design user verification methods for each phase

The "user verification method" is the heart of PXX. It must be an action a human **can actually perform** — not a technical metric:

**Poor examples (technical metrics):**
- ~~"All tests pass"~~
- ~~"API returns 200 status code"~~
- ~~"Database migration complete"~~

**Good examples (user actions):**
- "Enter test credentials on the login page and confirm successful entry to the dashboard"
- "Click the 'Export' button and confirm a correctly formatted CSV file can be downloaded"
- "Perform the original operations on the old page and confirm behavior is consistent with before the change" (for refactor phases)

Each verification action should include: what to do + what to expect to see.

---

## Step 5 — Draft the PXX document

Find `documents/planning/P00-planning-template.md` and read the template.

Confirm the next available PXX number (scan existing PXX files in `documents/planning/`).

Draft following the template structure, ensuring:

- **Background and motivation**: Explain why this change needs to be phased rather than going directly to FXX/BXX/RXX.
- **Overall goal**: One sentence, in user language, focused on the experiential change.
- **Impact scope**: List affected modules, annotating the change type for each module.
- **Overview table**: All phases visible at a glance, with initial status `[ ] Not started`.
- **Each phase card**:
  - Description is concise, explaining technical direction and logical boundary
  - User verification method is a specific checkbox list
  - Suggested document type (FXX / RXX / BXX) is filled in
  - Associated document field left as `—` (filled in after execution)
- **Handoff notes**: Do not modify; keep the template's original general guidance.

---

## Step 6 — Pre-submission checklist

After drafting is complete, verify each item:

- [ ] Does each phase leave the system in a runnable state?
- [ ] Is each "user verification method" an action a human can actually perform (not a technical metric)?
- [ ] Is each phase's boundary clear, not crossing two major functional areas?
- [ ] Does the phase order reflect real dependencies (later phases do not depend on earlier unfinished parts)?
- [ ] Is the "suggested document type" for each phase reasonable (new feature F, structural improvement R, defect fix B)?
- [ ] Are there any unconfirmed assumptions in the document? If so, have they been clearly annotated and flagged for user review?

---

## Submit for review

After completing the draft, explain to the user:

1. How many phases were planned and the approximate scale of the overall change
2. The core goal of each phase (one sentence)
3. Any assumptions or uncertain points that need user confirmation
4. Remind the user: **The review focus is on whether the phase breakdown is reasonable and whether the user verification methods are actionable** — not on technical details

Format:
```
PXX planning document draft complete.

N phases in total:
  P1 — [name]: [one-sentence goal] (suggested FXX)
  P2 — [name]: [one-sentence goal] (suggested RXX)
  ...

Assumptions needing confirmation:
  - [list uncertain points; omit if none]

Review focus:
  - Is the phase breakdown reasonable? Is any phase too large or too small?
  - Can you actually perform each "user verification method"?
  - Does the phase order match your execution plan?

Once approved, run /ddd-doc to start the first phase — the AI will read the PXX and draft the corresponding FXX / RXX / BXX document.
```

---

## During phase execution: updating PXX

After each phase's F/R/B document implementation is complete, the AI responsible for updating PXX should:

1. Fill in the "associated document" field for that phase with the actual file path (e.g., `documents/implements/F01-auth-login.md`)
2. Change that phase's status from `[~] In progress` to `[x] Completed`
3. Synchronously update the overview table
4. If this is the last phase, change the document frontmatter `status` to `completed`

**Do not mark a phase as `[x]` before implementation is complete (`ddd-tdd` acceptance has not yet passed).**
