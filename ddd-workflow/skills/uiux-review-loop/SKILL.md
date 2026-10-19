---
name: uiux-review-loop
description: Validate and improve a runnable interface, including newly implemented frontend acceptance in DDD, through a separate UIUX designer agent who uses the product, reports evidence-backed feedback, and retests after fixes until agreed acceptance criteria pass. Use for UIUX optimization, usability reviews with implementation, or a test-feedback-retest loop; not for creating a new product or reviewing screenshots alone.
---

# UIUX Review Loop

The main agent implements improvements. A separate UIUX designer agent operates the interface and decides whether the agreed user journeys pass. Code inspection and screenshots alone cannot establish usability approval.

## DDD integration

When invoked by `ddd-tdd`, this skill is its required frontend acceptance sub-loop. Use the approved FXX/RXX/BXX contract and return the verdict/evidence to the caller; do not start a nested `ddd-tdd`, mark the parent complete, or send completion notifications. The main agent makes in-scope fixes under existing approval and follows the caller's behavioral TDD rules. New frontend features enter once runnable; frontend refactors review retained journeys. A stopped non-pass review prevents parent completion.

When invoked directly in a DDD project, baseline observation may precede the spec, but code fixes still require the appropriate approved F/R/B. Reuse a matching active review record rather than spawning duplicate reviewers or resetting its budget.

## Establish the review contract

Read applicable project instructions, domain language, existing design conventions, and relevant approved requirements. Find the runnable entry point, startup command, available interaction tools, and any existing review record. Infer the target users and core journeys from the request and product; ask only about missing information that prevents meaningful testing.

Before the first test, record and tell the user:

- Target pages, user goals, and permitted change scope, including existing visual identity to preserve.
- Test environment, safe test data, and required viewports/input methods. For a responsive web app, normally include desktop and narrow mobile layouts plus keyboard operation of core flows; use product-specific targets for other interfaces.
- Concrete journey acceptance criteria: starting state, actions, and observable successful outcome. Include relevant empty, loading, validation, error, and completion states where they can be exercised.
- Pass gate and iteration budget. Default: at most **5 review rounds**, including the baseline. Honor a user-specified budget. Do not silently reset the counter or weaken acceptance criteria.

Create a compact record at `documents/uiux/<descriptive-slug>.md`, or the project's existing review location. Keep contract, issue ledger, round evidence, and latest status together. Resume an existing review without losing unresolved findings or prior rounds.

If the project requires approved F/R/B documents, review and evidence gathering can proceed immediately; implement only within existing authorization and approved scope. Capture uncovered scope in the appropriate document and resolve the actual approval requirement before that work. Do not request repeated approval for changes already covered. In a project without such a gate, a request to optimize authorizes reversible UI improvements within the stated scope.

## Launch the UIUX designer

Use the environment's subagent capability to launch **one separate UIUX designer/reviewer agent**. This skill explicitly calls for delegation. Use an internal subagent rather than creating a user-owned chat unless the user asks for a new chat. Do not choose a different model unless instructed.

Read [references/reviewer-brief.md](references/reviewer-brief.md) and adapt its brief with the concrete contract. Give the reviewer the URL/app entry point, test data, required journeys, viewports, acceptance criteria, and report location. The reviewer owns observation and verdicts; the main agent owns code changes. Do not give the reviewer the implementer's desired verdict or substitute an implementation summary for a test.

Keep the same reviewer across rounds using follow-up tasks. If a replacement is necessary, hand over the contract and ledger and require fresh operation of the latest UI. If delegation or actual interface control is unavailable, record `blocked`, report the missing capability, and stop the approval loop. A main-agent heuristic review may be useful but cannot be represented as this skill's independent pass.

## Test → feedback → fix → retest

1. **Test the current interface.** The reviewer actually performs the agreed journeys, observes layout and state transitions, and returns structured findings with evidence. Record the tested build/revision, working-tree state when relevant, environment, viewport, and round number. Ensure the running app reflects the current implementation.
2. **Triage feedback.** Use stable issue IDs. Each finding needs severity, user impact, reproduction steps, expected/observed behavior, and an evidence reference. Separate required corrections from optional design preferences. Resolve duplicate or unsupported findings through targeted observation, not a vote or an unverified dismissal.
3. **Fix within scope.** Address blockers and major friction first; preserve the design system and successful journeys. Record the change and affected issue IDs. Run relevant existing checks; add focused regression coverage when interaction behavior warrants it. Escalate unexplained test failures through the project's debugging workflow.
4. **Retest the actual result.** Send the reviewer the latest entry point/build and issue IDs ready for retest. The reviewer replays failed steps and affected journeys, then checks core-flow regressions. Only the reviewer can mark a finding verified fixed after observing the result. A proposed fix or passing code test is insufficient.
5. **Decide.** If the pass gate fails and rounds remain, continue automatically. If it passes, save the final reviewer evidence and report completion. Any change after that verdict invalidates approval for the affected scope and requires retesting.

Only one agent controls a shared browser/app at a time. The main agent must wait for the reviewer to finish before changing that shared session or implementation. Use isolated test sessions when available. Review authority does not grant permission for production purchases, destructive actions, messages, publishing, or external writes outside the user's authorization; use local/staging fixtures for those journeys and report untested portions accurately.

## Pass gate and stop conditions

Use `pass`, `needs-work`, or `blocked` as the current verdict:

- **pass:** Every required journey and criterion passed on the final build and required targets; no unresolved blocker or major issue; every required correction has been observed as fixed; the separate reviewer explicitly returned pass with evidence. Minor preferences can remain only if outside the agreed pass gate and listed in the final report. Do not claim WCAG conformance from this limited review.
- **needs-work:** A criterion failed or a required correction remains. Continue while budget remains. At the budget limit, report `needs-work`, remaining issues, and the next concrete step; never convert it to pass.
- **blocked:** A required journey cannot be exercised, tools/environment/access are missing, an actual approval is required, or a product decision cannot be inferred. Record the blocker and needed input. Two consecutive fix/retest rounds with the same unresolved issue and no observable improvement also stop for reassessment; do not consume unlimited rounds on cosmetic churn.

Keep criteria stable during a run. A user-approved scope change must be recorded and the affected criteria retested; it cannot retroactively turn an untested journey into a pass. Automated UX review is evidence about the specified tasks, not research with representative human users or a replacement for human acceptance required by the project.

## Review record and delivery

Use a compact ledger:

```text
Contract: scope | users | journeys/criteria | targets | pass gate | round limit
Status: pass / needs-work / blocked; current round / limit
Round: build/environment | journeys exercised | observations | evidence
Issues: ID | severity | impact/reproduction | expected/actual | evidence | fix | retest status
Handoff: reviewer agent | latest verdict | unresolved items | next action
```

Evidence may be screenshots, accessibility snapshots, interaction traces, or test logs appropriate to the observation; save durable references where supported and never invent paths or results. Avoid saving secrets or personal test data. At completion, report the verdict, rounds used, meaningful improvements, tested journeys/targets, remaining limitations or preferences, and link the record and evidence. Distinguish a blocked or budget-exhausted review from successful completion.
