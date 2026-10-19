---
name: ddd-tdd
description: DDD enhanced implementation skill. Driven by approved FXX / RXX / BXX documents, executes the red→green→acceptance TDD cycle. Integrates diagnose for unexpected failures, automatically requires independent UIUX review for affected frontend work before completion, and surfaces architectural observations after going green. Use when implementing features, refactoring, or fixing bugs.
---

# DDD TDD

The execution engine for all production code changes. Driven by approved implementation documents; does not accept vague chat instructions as a requirements source.

## Core rules

**No approved document, no code.**

If no FXX/RXX/BXX document is provided, stop and say: "Please provide the corresponding implementation document first, or run `/ddd-doc` to create one."

**Completion notification rules:**

- When `/ddd-tdd` completes a single FXX/RXX/BXX implementation standalone, send a completed notification via `ddd-email-notify`.
- If `/ddd-tdd` is called by a `/ddd-queue` worker, do NOT send a per-item completion notification upon finishing; only update the queue item and ledger. `/ddd-queue` sends one completed notification when the entire batch is done.
- To determine whether running inside a queue worker: if the prompt or context contains a Queue file / Item id / `you are a DDD queue worker`, or you are updating a QXX item status, treat yourself as a queue worker.
- If notification config is missing `notify_email_from` / `notify_email_to`, the email tool is unavailable, or the sender cannot be verified, report that no email was sent in the final summary but do NOT roll back the completed implementation.

---

## Implementation flow

### Prerequisites — Load context

**1. Read CONTEXT.md**
If `CONTEXT.md` exists in the root directory, read it and memorize the glossary. All domain terms used in test naming, behavior descriptions, and code comments must align with `CONTEXT.md`. If a test description uses a term not in the glossary, confirm the term's definition before proceeding.

**2. Read the implementation document**
Read and understand the provided FXX/RXX/BXX document. Extract:
- Feature goals and user stories
- Business rules and boundary conditions
- Acceptance criteria (listed item by item)
- Test scenario table (ID, Given, When, Then)
- Non-goals and known constraints
- UIUX applicability and reason, approved review journeys/criteria, targets, round budget, and review record path

Independently check frontend impact. New or changed user-facing frontend layout, content, interactions, navigation, responsive/accessibility behavior, or visible states require UIUX review; frontend refactors require usability regression review. Pure backend/non-UI work records not-required with a reason. For a missing or inconsistent contract, use `ddd-doc` to derive it from approved scope before frontend acceptance; request new approval only for an actual scope/behavior change.

If the document contains ambiguity, conflicts, or gaps, raise them **before implementation** and request clarification. If the user explicitly asks to proceed with assumptions, record the assumptions and continue, noting them in the implementation record.

**3. Inspect the codebase**
Identify:
- Language, framework, package manager
- Test framework and test execution command
- Existing test style and naming conventions
- Relevant production code and test files

---

### Red phase

**Goal: Have a failing test — failing for the right reason — before implementing behavior.**

Visual-only UIUX corrections use a recorded failing visual/usability observation followed by actual reviewer retesting; do not create artificial automated tests for styling. Behavior changes still follow the automated red → green steps below.

1. Map test scenarios from the document to concrete automated tests
2. For each test case, identify:
   - The linked acceptance criterion or business rule
   - The behavior under test
   - Required input conditions
   - Expected output or state change
3. Write the minimal failing test (start from the highest-priority test scenario)
4. Run only the target test and confirm it fails
5. Confirm the failure reason is "feature not yet implemented", not something else

**If the test does not fail:** Explain why and adjust the test until it fails for the right reason.

**Must not:**
- Skip the failing-test step for behavioral changes (unless the user explicitly asks to skip TDD)
- Weaken existing tests to make new tests pass
- Mock out the behavior under test itself

---

### Hook A — When test fails unexpectedly: structured debugging

**Trigger condition:** The test in the red phase fails, but the failure reason is **not** "feature not yet implemented" — instead it is an environment issue, dependency conflict, architectural constraint, or other confusing reason.

**Normal red** (continue to implementation):
> `AssertionError: expected undefined, got undefined` — feature logic not yet established

**Unexpected failure** (enter debugging loop):
> `TypeError: Cannot read property 'x' of undefined`, `Module not found`, `Connection refused`, problem with the test itself

**Execute the diagnose loop:**

1. **Establish a feedback loop** — find the fastest, reliably reproducible way to trigger the failure (failing test, CLI script, or minimal reproduction case)
2. **Generate hypotheses** — list 3-5 ranked hypotheses, each must be falsifiable:
   > "If X is the root cause, changing Y will make the failure disappear; changing Z will make the failure worse."
   Show the hypothesis list to the user before beginning validation.
3. **Validate one by one** — change only one variable at a time and compare observations against hypothesis predictions
4. **Fix and write a regression test** — after fixing, confirm the original red test is now a correct red

**Write diagnose results back to the FXX document:**
In the "Implementation record → Hypotheses and decisions" field, record:
- What unexpected failure occurred
- List of attempted hypotheses
- Confirmed root cause
- Fix strategy taken

Do not return from debugging to implementation until the problem has a clear root cause.

---

### Green phase

**Goal: Make the test pass with the minimum production code.**

1. Implement the minimum code needed to pass the test
2. Avoid large rewrites or unrelated cleanup
3. Do not change the public API unless the document requires it
4. Run the target test until it passes
5. If the change may affect neighboring behavior, run the related tests

Production code must implement the behavior described in the document, not merely satisfy the tests literally.

---

### Acceptance phase

Check off each item against the original FXX/RXX/BXX document:

- [ ] Every automatable test scenario in the document has a corresponding automated test
- [ ] Every acceptance criterion has appropriate verification evidence; visual/usability criteria also require the independent UIUX review below
- [ ] Every business rule is implemented
- [ ] Boundary conditions are handled
- [ ] Error scenarios are tested
- [ ] Implementation behavior is consistent with the document description (not approximate — consistent)
- [ ] If there are deviations, the reasons are recorded

If the implementation does not fully satisfy the document, explicitly report the gaps and **do not mark the feature as complete**.

---

### Required UIUX gate — when frontend experience is affected

After automated acceptance is green, invoke [uiux-review-loop](../uiux-review-loop/SKILL.md) automatically when the impact assessment requires it. This is part of `ddd-tdd` acceptance, not an optional suggestion and not a second implementation workflow.

1. Supply the approved F/R/B document, scoped journeys/criteria, runnable entry point, safe data, targets, round budget, and review record path. Launch the separate UIUX designer as specified by that skill. Newly built interfaces enter this gate once runnable; an existing baseline record may be resumed.
2. The designer operates the UI and reports evidence; the main agent fixes in-scope findings, reruns relevant automated checks, and requests actual retesting until the reviewer returns pass or a stop condition applies. Behavioral fixes follow red → green; visual polish uses observable UI evidence without artificial tests that mirror styling.
3. Stay inside this acceptance loop. Do not recursively call a second `ddd-tdd` or send per-round completion notifications. Amend and approve the document only when findings require new behavior or expanded scope; otherwise continue under existing approval.
4. Only an explicit reviewer `pass` on the final implementation satisfies the gate. Required journeys that are untestable, missing delegation/control tools, unresolved findings, or an exhausted budget prevent completion. Preserve the skill's actual `needs-work` / `blocked` verdict and evidence. An automated test pass cannot override it.
5. In a queue worker, a stopped non-pass review becomes queue item `blocked` with the underlying verdict and record link; hand control to the orchestrator and do not complete/commit the item or advance dependencies. In standalone/PXX work, keep the document partially implemented and the phase incomplete, reporting the needed next action.

For non-UI work, record `UIUX review: not-required` and the reason. For required work, both automated acceptance and UIUX pass must refer to the latest implementation; rerun affected verification after subsequent changes.

---

### Document update

After implementation is complete, update the "Implementation record" section of the FXX/RXX/BXX document:

```markdown
## Implementation record

### Status
Implemented / Partially implemented / Not implemented

### Implementation summary
Describe what was implemented.

### Test coverage
List newly added or updated tests and their corresponding document test case IDs.

### Changed files
#### Production code
- ...
#### Test code
- ...

### Acceptance criteria verification
| Acceptance criterion | Status | Basis |
|---|---|---|
| ... | Pass / Partial / Fail | Test name or description |

### Test scenario verification
| Test scenario ID | Status | Automated test basis |
|---|---|---|
| ... | Pass / Partial / Fail | Test name or description |

### Commands executed
```bash
...
```

### Hypotheses and decisions
List assumptions made during implementation and findings from the diagnose process (if any).

### UIUX acceptance
Applicability: required / not-required, with reason.
If required: pass / needs-work / blocked; rounds used / limit; tested build and targets; independent reviewer; review record/evidence link; unresolved findings.
Required non-pass means Partially implemented, not Implemented.

### Deferred items
List what was not implemented and why.

### Notes
Risks, follow-up recommendations, known constraints.
```

---

### Completion notification

After both the acceptance phase and document update are complete:

1. Determine whether this `/ddd-tdd` was called by a `/ddd-queue` worker.
2. If it is a queue worker:
   - Do not send a completed notification.
   - Record in the report and queue ledger: `ddd-tdd completed notification suppressed by ddd-queue; queue orchestrator will notify when all items complete.`
3. If it is not a queue worker:
   - Per `ddd-email-notify`, read `notify_email_from`, `notify_email_to`, `notify_on_tdd_completed` from `documents/ddd-email-notify.md`.
   - If `notify_on_tdd_completed: true` and From/To and the email tool are available, send a `[DDD TDD Completed]` notification.
   - If email cannot be sent, list the reasons in the final report.

Only send a completed notification when implementation status is "Implemented" and the required UIUX gate has passed (or is documented as not-required); do not send when partially implemented, blocked, tests not passing, or acceptance not complete.

---

### Hook B — After going green: surface architectural observations

After each significant green phase completes, ask the user:

> "Did the implementation expose any architectural issues? For example: overly tight module coupling, no suitable test seam, unclear responsibility boundaries — these are often signals for a potential RXX document."

- If yes: suggest running `/improve-codebase-architecture`; its output can serve directly as a candidate draft for a new RXX document
- If no: the flow ends; inform the user they can run `/ddd-doc` for document sync

**Long implementation cycles (after many conversation turns):**
If many turns have passed and the context is large, suggest:
> "The current session is very long. Consider running `/handoff` to preserve DDD state so you can continue in a new session."

---

## Safety rules

- Do not claim success without having run the tests (unless the environment cannot run tests and the reason is explicitly stated)
- Do not delete or weaken tests to make code pass
- Do not mock out the behavior under test itself
- Prefer behavior-level tests over implementation-detail tests
- Read the approved FXX/RXX/BXX before production changes. Behavioral changes require a failing test first; visual-only review corrections use the UIUX observation/retest evidence described above.
- Do not skip the post-implementation document update step
- Do not mark an FXX/RXX/BXX as complete before acceptance confirmation of coverage and the required independent UIUX pass

---

## Final output format

```markdown
## Changes

Production code:
- ...

Test code:
- ...

Updated documents:
- ...

## TDD evidence

Observed failing tests:
- ...

Observed passing tests:
- ...

Commands executed:
```bash
...
```

## FXX acceptance

Requirements document: [path]

Test scenario verification:
- ...

Acceptance criteria verification:
- ...

Implementation status: Implemented / Partially implemented / Not implemented

UIUX acceptance: required / not-required (reason); pass / needs-work / blocked when required; rounds and review record link.

Gaps or deferred items:
- ...

## Notes
Risks, skipped tests, assumptions, follow-up recommendations.

## Notification
- ddd-email-notify: sent / suppressed-by-queue / skipped-not-configured / failed
- From: <notify_email_from or —>
- To: <notify_email_to or —>
- Reason: <if email was not sent, explain why>
```
