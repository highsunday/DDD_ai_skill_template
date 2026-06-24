---
name: ddd-email-notify
description: Operational notification skill for DDD workflow. Used to display current email settings, configure or send ddd-tdd completed, queue blocked, and queue completed notification emails, maintain notify_email_from and notify_email_to in project config or QXX frontmatter, verify that the sending address is an authorized account in the current environment, and write to the Agent Communication Ledger after each send attempt (success or failure). Do not use for bulk marketing emails or general non-DDD notifications.
---

# DDD Email Notify

Handles operational notification emails for the DDD workflow. Primarily called by `ddd-tdd` and `ddd-queue`, but can also be used when the user asks to "configure notification email" or types `/ddd-email-notify` directly.

## Behavior when triggered directly

When the user types `/ddd-email-notify` directly without requesting to send an email or change settings, display current email info without sending:

```markdown
DDD Email Notify

Config source:
- <documents/ddd-email-notify.md or current QXX queue frontmatter; if neither exists, show "not configured">

Email settings:
- From: <notify_email_from or "not configured">
- To: <notify_email_to or "not configured">

When to send:
- `/ddd-tdd` standalone completed: will send (if notify_on_tdd_completed: true and From/To are available)
- `/ddd-queue` blocked: will send (if notify_on_queue_blocked: true and From/To are available)
- `/ddd-queue` all completed: will send (if notify_on_queue_completed: true and From/To are available)
- `/ddd-tdd` called internally by `/ddd-queue` worker completed: will not send, to avoid duplicate notifications per item

Current limitations:
- <whether the email tool is available; if cannot confirm, state that it will be verified before actual sending>
```

## Configuration fields

Project-level config file is located at:

```text
documents/ddd-email-notify.md
```

The project config or QXX frontmatter stores the following fields:

```yaml
notify_email_from: <sending address; must be an address authorized to send in the current environment>
notify_email_to: <destination address; notification recipient inbox>
notify_on_tdd_completed: true
notify_on_queue_blocked: true
notify_on_queue_completed: true
notify_email: <deprecated; legacy recipient address, migrate to notify_email_to>
```

Rules:

1. Do not store email passwords, tokens, SMTP keys, or app passwords.
2. `notify_email_from` is the sender identity, not the provider name.
3. `notify_email_to` is the recipient inbox.
4. Legacy QXX with only `notify_email` is treated as `notify_email_to`, and `notify_email_to` should be added on the next update.
5. If the user requests sending on completion or blocked, both `notify_email_from` and `notify_email_to` must be present.
6. QXX frontmatter takes priority over project config; missing fields can be filled in from `documents/ddd-email-notify.md`.
7. When `/ddd-queue` worker calls `/ddd-tdd` internally, `notify_on_tdd_completed` must be suppressed to avoid sending an email on each item completion.

## Configuration flow

When the user requests setting up email notifications:

1. If the user specifies a QXX, read the QXX frontmatter; otherwise read or create `documents/ddd-email-notify.md`.
2. If `notify_email_from` or `notify_email_to` is missing, ask the user to provide:
   - Sending address: `notify_email_from`
   - Destination address: `notify_email_to`
3. Validate both addresses meet basic email format: contains one `@`, and both local and domain parts are non-empty.
4. Configure when to send:
   - `notify_on_tdd_completed: true`
   - `notify_on_queue_blocked: true`
   - `notify_on_queue_completed: true`
   Unless the user explicitly disables any of them.
5. Update QXX frontmatter or `documents/ddd-email-notify.md`.
6. Do not test sending unless the user explicitly requests it.

## Pre-send checks

Before sending, confirm:

1. `notify_email_to` exists.
2. `notify_email_from` exists.
3. The current environment has an available email tool, such as a Gmail connector, a project-provided notification script, `sendmail`, or `mail`.
4. If the tool can only send from the currently logged-in account, confirm that account matches `notify_email_from`.
5. If the sending address cannot be verified, do not send; instead report the current blocked or completed status in the current conversation.

## When to send

### ddd-tdd standalone completed

Send when `/ddd-tdd` completes a standalone F/R/B implementation, conditions:

1. Implementation status is `implemented`.
2. Red, green, acceptance, and documentation update are all complete.
3. Not called by a `/ddd-queue` worker.
4. `notify_on_tdd_completed: true`.
5. `notify_email_from` and `notify_email_to` are available and the sending address passes verification.

If `/ddd-tdd` is called by a `/ddd-queue` worker, never send a per-item completion notification; the queue orchestrator sends one notification when the entire batch is complete.

### ddd-queue blocked

Send when a `/ddd-queue` item is blocked, conditions:

1. The queue has stopped.
2. `notify_on_queue_blocked: true`.
3. `notify_email_from` and `notify_email_to` are available and the sending address passes verification.

### ddd-queue completed

Send when all `/ddd-queue` items are complete, conditions:

1. All non-skipped items in the queue are `completed`.
2. No pending / in_progress / blocked items remain.
3. The queue did not merely pause after reaching a batch limit.
4. `notify_on_queue_completed: true`.
5. `notify_email_from` and `notify_email_to` are available and the sending address passes verification.

## Email content

Blocked notification format:

```text
From: <notify_email_from>
To: <notify_email_to>
Subject: [DDD Queue Blocked] <item id> <item title> needs confirmation

Completed:
- <item id> <title>, commit: <hash>

Currently blocked:
- <item id> <title>

Blocker reason:
<blocker_reason>

Your confirmation is needed:
- A: ...
- B: ...

Queue is paused; subsequent items have not been executed.
```

ddd-tdd completed notification format:

```text
From: <notify_email_from>
To: <notify_email_to>
Subject: [DDD TDD Completed] <F/R/B id> <title>

Completed:
- Document: <path>
- Implementation status: implemented

Tests:
- <command>: <result>

Change summary:
<summary>
```

queue completed notification format:

```text
From: <notify_email_from>
To: <notify_email_to>
Subject: [DDD Queue Completed] <QXX title>

Queue fully completed:
- <QXX-01 title>, commit: <hash>
- <QXX-02 title>, commit: <hash>

Tests:
- <main command and result>

Queue document:
<path>
```

## Delivery result

After each notification attempt, return and write to the `Agent Communication Ledger`:

```md
#### LXXX — <time> — <item> — orchestrator -> user — notification

**Message**
DDD queue blocked notification.

**Context**
- From: <notify_email_from>
- To: <notify_email_to>
- Subject: <subject>

**Artifacts**
- Delivery status: sent | draft-created | failed | fallback-dialog
- Tool: <tool name or unavailable>

**Follow-up**
- <if failed, explain what the user needs to do to fix the configuration>
```

If the queue email fails or can only create a draft, do not proceed to the next queue item. If a standalone `/ddd-tdd` email fails, report the notification failure in the final summary but do not roll back the completed implementation.
