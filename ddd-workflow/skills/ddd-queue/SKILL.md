---
name: ddd-queue
description: "DDD long-running work queue skill. Creates or executes QXX queue documents under documents/queue/, suitable for 2 to 5 ordered, auto-advanceable feature, bug fix, or refactor tasks. Before creating a queue, a centralized grill-me must be run to clarify requirements, design questions, dependencies, acceptance criteria, and stop conditions for all items; the QXX must be updated before any item may be executed. Each item must be handled by a new Codex or Claude Code session, fully running ddd-start, ddd-doc, and ddd-tdd, then git committed, with all dispatch, questions, answers, decisions, tests, and handoffs recorded in the Agent Communication Ledger between the two agents, orchestrator, and user. On blocker, send email to notify the user and stop; send a completed notification when the entire batch finishes. ddd-tdd called internally by a queue worker must not send a per-item completion notification. Do not use for a single task, for work where requirements still need human stage-by-stage decisions, or for work where the scope of later stages cannot be defined until an earlier stage completes — use ddd-doc/ddd-tdd or ddd-plan for those cases instead."
---

# DDD Queue

Manages a batch of ordered, auto-advanceable DDD tasks, aiming to extend the period during which AI can work autonomously and reduce per-item interruptions to the user. `ddd-queue` handles both independent items and explicitly dependent sequential items; `ddd-plan` handles large changes that still require planning, decomposition, or human review.

How to decide:

- Use `ddd-queue`: the user can already list 2 to 5 specific items, each with requirements, acceptance criteria, and stop conditions; if there are dependencies, the dependency and unlock condition can be confirmed from documents, commits, or test results.
- Use `ddd-plan`: the scope of later items is unknown until an earlier stage completes, there are architectural route choices, or the stage decomposition itself requires human review.

## Core rules

1. Only let one new session handle one queue item at a time.
2. Do not implement multiple items consecutively in the same Codex / Claude Code session.
3. Every completed item must produce one git commit containing code, tests, F/R/B documents, and a queue status update.
4. Execute at most 5 items per run; default to at most 3 when unspecified.
5. On a blocker, stop the entire queue immediately — do not continue to the next item.
6. Auto-approve applies only to clear, low-risk, verifiable items; ambiguous or high-risk items must be blocked.
7. The working tree must be clean before executing the queue to prevent auto-commits from including uncommitted user changes.
8. Dependent items must declare `depends_on` and `unlock_condition`; the queue must be blocked when an unlock condition cannot be verified.
9. Every worker sub-session must treat the assigned item as a new DDD task and fully execute `ddd-start → ddd-doc → ddd-tdd` — direct code modification is not permitted.
10. Sub-sessions do not conduct real-time conversations; when user input is needed, write to the item's blocked / questions fields, and the orchestrator notifies the user and stops.
11. The queue document is the shared communication ledger for Codex, Claude Code, the orchestrator, and the user; all cross-session messages must be appended to the `Agent Communication Ledger`.
12. Communication records are append-only: existing log entries must not be deleted or overwritten; if a prior entry is incorrect, append a correction / superseded entry.
13. When creating a queue, a centralized `grill-me` must be run first to clarify requirements, design risks, dependencies, acceptance criteria, and stop conditions for all items in one session.
14. No worker may be executed before the queue's centralized clarification is complete; at execution time, `intake_grill_status: completed` must be confirmed and every item to be executed must be `clarification_status: clarified`.
15. The QXX main document retains only the summary, index, and unresolved questions needed for execution; long stdout, full conversations, and historical details are archived to `documents/queue/logs/` to avoid repeated token consumption by later workers.
16. If the queue is configured to send email on blocked or batch-completed events, `notify_email_from` and `notify_email_to` must be explicitly set in the QXX frontmatter or `documents/ddd-email-notify.md`; email sending and settings validation are delegated to the `ddd-email-notify` rules — email passwords, tokens, SMTP keys, or app passwords must not be written into the QXX.
17. `ddd-tdd` called internally by a queue worker must not send a per-item completion notification; only the queue orchestrator sends one completed notification when the entire batch finishes.

## Queue document location

Queue documents are placed in the project root:

```text
documents/queue/
├── Q00-queue-template.md
└── Q01-<batch-name>.md
```

If `documents/queue/Q00-queue-template.md` does not exist, copy it from `ddd-create-folder/templates/Q00-queue-template.md`, or create it following the format defined in this skill.

## Queue Item format

Each item must have the following fields:

```md
## Q01-01 <feature name>

id: Q01-01
type: FXX
agent: codex
status: pending
clarification_status: pending
depends_on: []
unlock_condition: none
auto_approve: true
commit_required: true
implemented_doc: —
commit: —
worker_session: —
worker_log: —
handoff_summary: —
communication_entries: []

### Requirements
<clear description of the user behavior to be completed>

### Acceptance criteria
- [ ] <user-operable confirmation action and expected result>

### Stop conditions
- <when to block rather than guess>

### Blocker log
None yet.
```

Field rules:

- `type`: `FXX` / `BXX` / `RXX`
- `agent`: `codex` / `claude` / `auto`
- `status`: `pending` / `in_progress` / `completed` / `blocked` / `skipped`
- `clarification_status`: `pending` / `clarifying` / `clarified` / `blocked`; only `clarified` items may enter a worker
- `depends_on`: list of dependent item ids, e.g. `[Q01-01]`; use `[]` when there are no dependencies
- `unlock_condition`: when this item may begin, e.g. `Q01-01 completed and login page can submit normally`
- `auto_approve`: `true` means the worker may create the F/R/B document and treat it as approved; however, the worker must still wait until the document is clear before modifying code.
- `implemented_doc`: fill in the actual document path after completion, e.g. `documents/implements/F03-user-search.md`
- `commit`: fill in the commit hash after completion
- `worker_session`: the orchestrator may fill in `codex` / `claude` and a timestamp when starting a sub-session to aid tracking
- `worker_log`: fill in a summary, test commands, risks, and limitations when the worker completes or is blocked
- `handoff_summary`: a short handoff summary for the next agent, including current state, key files, decisions, and risks
- `communication_entries`: list of `Agent Communication Ledger` entry ids associated with this item, e.g. `[L001, L004]`

## Email Notification Settings

QXX supports blocked and batch-completed notification emails. QXX frontmatter can override the project config's email settings:

```yaml
notify_email_from: <optional, sending address; must be an authorized sender in the current environment>
notify_email_to: <optional, notification recipient address>
notify_on_queue_blocked: true
notify_on_queue_completed: true
notify_email: <deprecated, optional; legacy blocked-notification recipient address, use notify_email_to instead>
```

Settings rules:

1. `notify_email_from` is the sending address, not a password or provider configuration.
2. `notify_email_to` is the recipient address to notify when blocked or completed.
3. `notify_email` is retained only for legacy document compatibility; if a QXX has only `notify_email`, the orchestrator treats it as `notify_email_to` and adds `notify_email_to` on the next QXX update.
4. If the user requests queue notifications, `notify_email_from` and `notify_email_to` must be asked for and filled in when creating or updating the QXX, or confirmed to already be available in `documents/ddd-email-notify.md`.
5. If the user does not request email sending, both fields may be left empty; on blocked, report directly in the current conversation.
6. If the email tool does not support specifying a From address (e.g., it can only send from the currently logged-in account), the orchestrator must confirm that the tool's authorized account matches `notify_email_from`; if they do not match, email must not be sent — the queue must be stopped and the issue reported in the current conversation.
7. The QXX does not store any email credentials. Email capability is provided by the current Codex / Claude / MCP / CLI execution environment.
8. Settings, validation, and email sending details are handled by `ddd-email-notify`.
9. Every email send or send failure must append a ledger entry; the recommended types are `notification` or `blocked`.
10. When the queue stops after reaching `batch_limit` with items still pending, that does not count as queue completed and no completed notification is sent.

## Centralized requirement clarification (Queue Intake Grill)

The core goal of `ddd-queue` is to let the user handle all major uncertainties in one session before execution, avoiding repeated interruptions from each worker sub-session.

When creating or updating a QXX, run a centralized `grill-me` first:

1. Treat the entire batch of queue items as a single intake session, not as separate per-item sessions.
2. Check each item for:
   - User goals and observable behavior
   - Whether acceptance criteria can be confirmed by a human action
   - Whether automated tests can be derived
   - Inter-item dependencies and `unlock_condition`
   - UI / API / data model / permissions / migration / security risks
   - Non-goals and stop conditions
   - Which unanswered questions would cause a worker to block
3. Consolidate all questions into "Queue Intake Questions" grouped by item, then ask the user all at once.
4. After the user responds, update the QXX:
   - frontmatter `intake_grill_status: completed`
   - frontmatter `ready_for_execution: true`
   - Each executable item `clarification_status: clarified`
   - Each item's requirements, acceptance criteria, stop conditions, dependencies, unlock conditions, and design notes
   - Append `intake-question`, `answer`, and `decision` entries to the `Agent Communication Ledger`
5. If there are still unanswered blocking questions, keep the queue at `ready_for_execution: false`, set the corresponding items to `clarification_status: blocked`, and do not start any workers.

The output of the centralized grill-me must be written to the "Queue Intake Review" section of the QXX. Recommended structure:

```md
## Queue Intake Review

intake_grill_status: completed
ready_for_execution: true
reviewed_by: codex
reviewed_at: <YYYY-MM-DD HH:MM>

### Clarification Matrix

| Item | Clarification | Key Decisions | Remaining Questions |
|------|---------------|---------------|---------------------|
| Q01-01 | clarified | <decision summary> | — |

### Queue Intake Questions

#### Q01-01
- Q: <question>
- A: <user answer>

### Cross-item Decisions

- <cross-item dependencies, ordering, and shared design decisions>
```

Worker sub-sessions must not re-run an interactive grill-me for items with `clarification_status: clarified`. If a worker discovers a new contradiction not covered by the QXX during implementation, it must block, append a `question` entry, and let the orchestrator stop the queue and ask the user.

## Context Budget / Token Policy

`ddd-queue`'s default strategy is "short main document, queryable history, worker reads precisely." The QXX is a work-state index, not a full verbatim transcript.

The main document retains:

- frontmatter, overview, and Queue Intake Review summary
- Each item's requirements, acceptance criteria, stop conditions, and current status
- At most 8 lines of `handoff_summary` per item
- The `Log Index` of the Agent Communication Ledger
- Recent, unresolved, or active entries the next worker must read

The main document does not retain:

- Full stdout / stderr
- Full test output
- Long worker replies
- Historical details from completed items that no longer affect subsequent items

Archiving rules:

1. Long content goes into `documents/queue/logs/<QXX>-<entry-id>.md` or `documents/queue/logs/<QXX>-archive.md`.
2. The `Log Index` in the QXX retains the `Archive Ref` path.
3. A single ledger entry should not exceed 1200 words; if it does, keep a summary and archive the full text.
4. A single `worker_log` should not exceed 8 lines; test output lists only the command, result, and key errors.
5. A single `handoff_summary` should not exceed 8 lines; write only the state, files, decisions, tests, and risks the next agent must know.
6. When the QXX exceeds approximately 500 lines, the ledger exceeds 20 entries, or completed items have more than 3 entries each, perform compaction first: retain the index, status, decisions, unresolved questions, and handoff, then archive older details.

When reading context, a worker reads only:

- QXX frontmatter, overview, and Queue Intake Review
- The assigned item's section
- `handoff_summary`, `implemented_doc`, and `commit` of dependent items
- `Log Index`
- Active entries listed in `communication_entries` for the assigned item and its dependencies

Unless a blocker, contradiction, or acceptance check requires it, workers do not read the full archive.

## Agent Communication Ledger

Every QXX must have a global `Agent Communication Ledger`. This is the single source of truth for communication between two agents, orchestrator dispatch, user answers to questions, and after-the-fact decision tracing.

Ledger principles:

1. **Append-only**: only new entries may be appended; existing entries must not be deleted or overwritten.
2. **Readability first**: record operation summaries, questions, answers, decisions, test evidence, and handoff information that a user can read after the fact.
3. **No hidden reasoning**: do not store private reasoning drafts; store verifiable judgments, assumptions, rationale, and conclusions.
4. **Record both directions**: orchestrator → worker dispatch, worker → orchestrator status, worker → user questions, and user → worker answers all require entries.
5. **Summarize long output**: when stdout / test output is too long, record a summary and key errors; if there is an external log file, record its path.
6. **Index first**: later agents read the `Log Index` first, then open only entries / archives relevant to the current item, its dependencies, or unresolved questions.

Recommended format:

```md
## Agent Communication Ledger (Append-only)

### Log Index

| Entry | Time | Item | From -> To | Type | Summary | Archive Ref |
|-------|------|------|------------|------|---------|-------------|
| L001 | 2026-05-23 10:00 | Q01-01 | orchestrator -> codex | dispatch | Dispatch Q01-01 | — |

### Active Entries

#### L001 — 2026-05-23 10:00 — Q01-01 — orchestrator -> codex — dispatch

**Message**
Dispatch Q01-01. Process with ddd-start -> ddd-doc -> ddd-tdd, then commit when done.

**Context**
- Queue file: `documents/queue/Q01-example.md`
- Depends on: []
- Unlock condition: none

**Expected Response**
- Update item status
- Create F/R/B document
- Record red and green tests
- commit hash

**Artifacts**
- —

**Follow-up**
- —

**Archive**
- —
```

Entry `Type` recommended values:

- `dispatch`: orchestrator assigns a worker
- `intake-question`: batch requirement clarification questions raised during the centralized grill-me phase
- `status`: worker reports current progress
- `ddd-start`: worker completes requirement type determination
- `ddd-doc`: worker creates or updates an F/R/B document
- `tdd-red`: worker records red tests
- `tdd-green`: worker records green tests and acceptance
- `question`: worker needs a user answer
- `answer`: user or orchestrator responds
- `decision`: records an adopted product or technical decision
- `handoff`: handoff summary for the next agent
- `blocked`: worker is blocked and stopped
- `notification`: orchestrator has sent a blocked notification, or reports in conversation after a send failure
- `completed`: worker completes an item
- `correction`: corrects a previous entry
- `archive`: moves old entries or long output to archive
- `compaction`: compacts the QXX main document, retaining index, status, and handoff

## Creating a Queue

When the user asks to queue a batch of tasks, extend AI autonomous work time, or reduce per-item interruptions:

1. Read `CONTEXT.md` and use the existing domain language.
2. Create `documents/queue/` and confirm the next available `QXX` number.
3. Create a QXX draft first, with frontmatter set to:
   - `status: draft`
   - `intake_grill_status: pending`
   - `ready_for_execution: false`
4. Organize each requirement into an item draft; each item starts with `clarification_status: pending`.
5. Run a centralized `grill-me` to clarify all items in one session.
6. After the user responds, fully update the QXX's Queue Intake Review, item requirements, acceptance criteria, stop conditions, dependencies, and design notes.
7. If there are dependencies between items, write `depends_on` and `unlock_condition`, and sort items by dependency order.
8. If the requirements for a later item cannot be known until an earlier item completes, stop and suggest using `ddd-plan` instead.
9. If an item is too ambiguous, mark it as `clarification_status: blocked` and do not put it into an executable state.
10. Fill in the queue frontmatter:
   - `status: ready` (only when all items to be executed are clarified)
   - `intake_grill_status: completed`
   - `ready_for_execution: true`
   - `batch_limit: 3`, unless the user specifies a value from 1 to 5
   - `notify_email_from`, if the user requests queue notifications
   - `notify_email_to`, if the user requests queue notifications
   - `notify_on_queue_blocked: true`
   - `notify_on_queue_completed: true`
   - `notify_email`, only when migrating a legacy QXX as a compatibility alias — do not add it to new documents

After completion, report the queue document path and item list. If the user also requests execution, continue to "Executing a Queue."

## Executing a Queue

The AI's current session is the orchestrator. The orchestrator is only responsible for selecting items, starting new sub-sessions, checking results, sending email, and stopping; do not implement features directly in the orchestrator session.

### Worker Session Communication Model

Workers started via `codex exec` or `claude -p` are non-interactive sub-sessions. The orchestrator can read their stdout / stderr and final results, but must not assume it can reliably inject new questions or correct direction mid-execution.

The default communication method is the document protocol:

1. Before starting a worker, the orchestrator appends a `dispatch` entry and writes the entry id into the item's `communication_entries`.
2. After starting, the worker first reads the `Log Index`, the assigned item's active `communication_entries`, and all dependent items' `handoff_summary`.
3. When the worker reaches key milestones in ddd-start / ddd-doc / ddd-tdd, it appends the corresponding entry.
4. When the worker needs a human decision, it sets the item to `status: blocked`, appends a `question` or `blocked` entry, and fills in `blocker_reason`, `questions`, or `need_user_decision` in the item.
5. The orchestrator detects the blocked state, sends email or reports to the user, and stops the queue.
6. After the user provides additional answers, the orchestrator appends an `answer` entry and starts a new worker session to continue.

If truly real-time two-way communication is needed, use an interactive tmux / remote-control style runner instead; however, this reduces queue reproducibility and is not the default process.

### Step 1 — Pre-flight checks

1. Read the queue document.
2. Confirm centralized clarification is complete: `ready_for_execution: true`, `intake_grill_status: completed`, and all items to be executed in this run are `clarification_status: clarified`.
3. If centralized clarification is not complete, stop execution and return to "Centralized requirement clarification (Queue Intake Grill)"; do not start any workers.
4. Confirm `git status --short` is clean. If not, stop and ask the user to handle it first; do not auto-commit the user's existing changes.
5. If the QXX has `notify_email_from`, `notify_email_to`, or the legacy field `notify_email` set, validate the email notification settings per `ddd-email-notify`:
   - If only the legacy field `notify_email` is present, treat it as `notify_email_to` and add the new field on the next QXX update.
   - If the user requires email sending but `notify_email_from` or `notify_email_to` is missing, stop and require them to be filled in.
   - Confirm a usable email tool is available in the current environment; if not, execution may continue, but blocked events can only be reported in the current conversation.
   - If the email tool does not support specifying a From address, confirm the currently authorized sending account matches `notify_email_from`; if they do not match, stop and require the user to correct the sending address or switch the authorized account.
6. Confirm whether `codex` / `claude` CLI is available:
   - Codex: `codex exec`
   - Claude Code: `claude -p`
7. Determine the maximum number of items to process in this run: use the user-specified value, then the queue's `batch_limit`, then default to 3; never exceed 5.

### Step 2 — Select the next item

In document order, select the first `status: pending` item whose dependencies are all completed.

Dependency check:

1. Read the item's `depends_on`.
2. Confirm every dependent item is `status: completed` and has a non-empty `commit`.
3. Check whether the `unlock_condition` can be reasonably confirmed from the queue, F/R/B documents, test results, or current code state.
4. If a dependent item is not yet complete, process the earliest incomplete dependency first.
5. If a dependent item is blocked, has a missing commit, or the unlock condition cannot be verified, block the current item and stop the queue.

If `agent: auto`:
- If both CLIs are available, prefer to alternate following the strategy used by the previous item in the queue.
- If only one CLI is available, use the available one.
- If no CLI is available, block the item and stop.

### Step 3 — Start a new worker session

Every item must use a new CLI process. Do not use `codex resume`, `claude --continue`, or `claude --resume`.

Codex worker example:

```bash
codex exec \
  --cd "$REPO_ROOT" \
  --sandbox danger-full-access \
  --ask-for-approval never \
  --ephemeral \
  "$WORKER_PROMPT"
```

Claude Code worker example:

```bash
claude -p \
  --no-session-persistence \
  --permission-mode bypassPermissions \
  "$WORKER_PROMPT"
```

If an explicit working directory is needed in Claude Code, set the shell command's working directory to the project root before running `claude -p`.

### Worker Prompt

The prompt given to a sub-session must be self-contained and include at least the following:

```text
You are a DDD queue worker. Handle only one queue item; do not handle any other items.

Repo: <project root>
Queue file: <queue document path>
Item id: <QXX-YY>

You must follow these rules:
1. Read the ddd-queue rules; if the skill is unavailable, follow this prompt instead.
2. Read the queue document and handle only the assigned item.
3. Confirm the queue has completed centralized clarification: ready_for_execution: true, intake_grill_status: completed, assigned item clarification_status: clarified. If not, block — do not implement.
4. Before starting, confirm the working tree is clean; if not, report blocked — do not commit.
5. Check depends_on; all dependent items must be completed with a commit, and unlock_condition must be verifiable.
6. Read the Queue Intake Review, Log Index, the assigned item's active communication_entries, dependent items' handoff_summary, implemented_doc, commit summary, and test records to build the context needed for this item; do not read the full archive unless blocked, contradictions arise, or acceptance requires it.
7. Mark the assigned item as in_progress, fill in worker_session, and append a status entry.
8. Execute ddd-start for the assigned item as a new task: determine whether it is FXX / BXX / RXX and load CONTEXT.md; if ddd-start would request a grill-me, cross-check against the Queue Intake Review — if already answered, continue; if not, block. Append a ddd-start entry.
9. Execute ddd-doc: create or update the corresponding FXX / BXX / RXX document; the document must include requirements, acceptance criteria, test scenarios, stop conditions, and assumptions. Append a ddd-doc entry.
10. If auto_approve: true and the document produced by ddd-doc is clear, low-risk, and testable, treat that document as the approved document for this item; otherwise block, append a question / blocked entry, and wait for user review.
11. Execute ddd-tdd: first write failing tests and confirm red, append a tdd-red entry; then implement the minimal green, then run acceptance and update the document implementation record, append a tdd-green entry. Because this ddd-tdd is executed inside a ddd-queue worker, suppress the ddd-tdd per-item completion notification — do not send email.
12. Do not skip red tests; if a reasonable failing test cannot be created, block and explain why.
13. If requirements are ambiguous, high-risk, tests cannot pass stably, or a user decision is needed, mark the item as blocked, fill in blocker_reason, questions, or need_user_decision, append a question / blocked entry, and stop.
14. On completion, update the queue item: status: completed, implemented_doc, commit, worker_log, handoff_summary, communication_entries, and append a handoff / completed entry.
15. The git commit must include only files related to this item; use an explicit git add file list — do not use git add .
16. After committing, confirm the working tree is clean.
17. If the QXX exceeds the context budget, perform compaction first, archive long logs, and append a compaction / archive entry.
18. The final reply reports only the item status, commit hash, test commands run, ledger entries, ddd-tdd completed notification suppressed by queue, and any limitations. Do not start the next item.
```

## Worker DDD flow

The worker sub-session must execute the following steps in order without exception:

1. **ddd-start**: treat the queue item as a new task from the user, determine the type, load `CONTEXT.md`, and check whether prerequisite flows such as grill / zoom-out / diagnose are needed. If the requirements are not suitable for auto-approval, block.
2. **ddd-doc**: create or update one FXX / BXX / RXX document. The queue item is a dispatch source, not an implementation document; the true requirements authority is still the F/R/B document.
3. **ddd-tdd**: use the newly created F/R/B document as the sole requirements source; run red, green, acceptance, and documentation updates. Because this is executed inside a queue worker, do not send the ddd-tdd per-item completion notification.
4. **queue closeout**: update the queue item's status, associated documents, test commands, commit hash, and worker log.
5. **communication closeout**: append `handoff` / `completed` entries and update the item's `handoff_summary` and `communication_entries`.

If a worker finds that it has already started modifying production code before an F/R/B document exists, it must stop, revert the incomplete changes for this item, and return to ddd-doc first.

## Worker completion rules

Before a worker completes an item it must have:

1. Recorded the ddd-start determination result in the worker log.
2. Created or updated one F/R/B document and written the path into `implemented_doc`.
3. Added or updated tests based on that document.
4. Observed and recorded at least one red test that fails for the correct reason.
5. Run the relevant tests, and broader regression tests if necessary.
6. Updated the F/R/B document's implementation record.
7. Updated the queue item status, overview table, and worker log.
8. Updated `handoff_summary` so the next agent does not need to re-read the full stdout.
9. Appended the necessary ledger entries: at minimum dispatch, ddd-start, ddd-doc, tdd-red, tdd-green, and handoff or blocked; long content retains only a summary with an archive ref.
10. Created a git commit.
11. Confirmed `git status --short` is clean.
12. If the context budget threshold is triggered, performed compaction to ensure the next worker does not need to read the full history.

Recommended commit message format:

```text
feat: complete Q01-01 <feature name>
fix: complete Q01-02 <fix name>
refactor: complete Q01-03 <refactor name>
```

## Blocker rules

The following situations must be blocked:

- Requirements lack verifiable acceptance behavior.
- The queue has not completed centralized grill-me, or the item's `clarification_status` is not `clarified`.
- The item's dependencies are undeclared, a dependent item is blocked, a dependency commit is missing, or `unlock_condition` cannot be verified.
- Changes to data models, permissions, payments, security, authentication, privacy, data deletion, or migrations are required but not explicitly authorized in the queue.
- The root cause cannot be identified after two consecutive test failures.
- The user must choose a product behavior or trade-off.
- The working tree is not clean, which may include changes not belonging to this item.
- The specified page, module, API, or test entry point cannot be found and cannot be reasonably inferred from the repo.

When blocked, update the item:

```md
status: blocked
blocker_reason: <specific reason>
questions:
- <questions the user needs to answer>
need_user_decision:
- A: <option>
- B: <option>
worker_log: <which steps were completed, where it got stuck, whether there are uncommitted changes>
communication_entries: [<related LXXX>]
```

Do not commit incomplete production code. If partial changes have been produced and cannot be safely completed, leave a clear record and stop; the orchestrator must not continue to the next item.

## Notifying the user

After the orchestrator detects a blocked state:

1. Stop the queue.
2. Read `notify_email_from` and `notify_email_to` from the queue frontmatter per `ddd-email-notify`.
3. If a legacy document has only `notify_email`, treat it as `notify_email_to` and add `notify_email_to` on the next QXX update.
4. If `notify_on_queue_blocked: false`, do not send email — report the blocked state directly in the current conversation.
5. If there is no `notify_email_to`, no usable email tool, or the sending address cannot be verified, report the blocked state directly in the current conversation.
6. If the email tool does not support specifying a From address, confirm the currently authorized account matches `notify_email_from`; if they do not match, do not send email and report in the current conversation instead.
7. Whether email sending succeeds or fails, append a `notification` ledger entry recording From, To, subject, delivery status, and whether the fallback to conversation reporting was used.
8. A send failure must not allow continuing to the next item.

Email format:

```text
From: <notify_email_from>
To: <notify_email_to>
Subject: [DDD Queue Blocked] <item id> <item title> — confirmation needed

Completed:
- <item id> <title>, commit: <hash>

Currently blocked:
- <item id> <title>

Blocker reason:
<blocker_reason>

Needs your confirmation:
- A: ...
- B: ...

The queue has been paused; subsequent items have not been executed.
```

## Queue completion notification

After the orchestrator completes each item, it checks whether the entire queue is done. The queue is considered completed only when all of the following conditions are met simultaneously:

1. All non-skipped items are `status: completed`.
2. There are no `pending`, `in_progress`, or `blocked` items.
3. The queue was not paused because it reached `batch_limit`.
4. Every completed item has a `commit` and an `implemented_doc`.

After queue completion:

1. Stop the queue.
2. Read `notify_email_from`, `notify_email_to`, and `notify_on_queue_completed` per `ddd-email-notify`.
3. If `notify_on_queue_completed: true` and email settings are available, send a `[DDD Queue Completed]` notification.
4. Whether email sending succeeds or fails, append a `notification` ledger entry recording From, To, subject, delivery status, and whether the fallback to conversation reporting was used.
5. If no usable email settings or tool is available, report queue completion directly in the current conversation and explain why email was not sent.

Email format:

```text
From: <notify_email_from>
To: <notify_email_to>
Subject: [DDD Queue Completed] <QXX title>

Queue fully completed:
- <item id> <title>, commit: <hash>

Tests:
- <main commands and results>

Queue document:
<queue file path>
```

## Final report

When the orchestrator stops, it reports:

```markdown
DDD queue execution finished.

Completed:
- Q01-01 <title> — <commit>
- Q01-02 <title> — <commit>

Blocked:
- Q01-03 <title> — <reason>

Not executed:
- Q01-04 <title>

Tests:
- <main commands run per item>

Communication records:
- Agent Communication Ledger: L001-L0XX

Notifications:
- ddd-email-notify: sent / skipped-not-configured / failed / not-completed
- From: <notify_email_from or —>
- To: <notify_email_to or —>
- Reason: <if email was not sent, explain why>

Queue document:
- documents/queue/Q01-xxx.md
```
