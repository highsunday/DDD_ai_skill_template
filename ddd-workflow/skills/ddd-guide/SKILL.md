---
name: ddd-guide
description: Create or update GXX operation guides under documents/guides/. These documents record how to run tests, start services, build packages, and other operations in a "quick commands first, details after" format, so engineers do not need to ask AI every time. Use when an engineer explicitly says "write me a guide" or "create a guide".
---

# DDD Guide

Create or update GXX operation guides under `documents/guides/`.

## Document positioning

A GXX guide is an "operation manual for engineers to look up", not a feature spec or architecture description:

| Type | Folder | Audience | Purpose |
|------|--------|----------|---------|
| FXX / RXX / BXX | `implements/` | AI + Engineers | Record requirements and implementation decisions |
| Module docs | `modules/` | AI + Engineers | Understand architecture and responsibility boundaries |
| **GXX guide** | **`guides/`** | **Engineers** | **Look up operation commands without asking AI** |

One guide corresponds to one type of operation (testing / starting / building / deploying, etc.), **not every feature needs one** — only record frequently used operations.

---

## Step 1 — Confirm operation scope

Ask the user (if not stated in the command):

1. **What operation should this guide record?** (run tests / start service / build package / other)
2. **For which service or module?** (e.g., backend, frontend, CLI tool)
3. **What is the most commonly used one-line command?** (this will become the quick command at the very top of the guide)

If the user has already provided this information in the command, proceed to the next step directly.

---

## Step 2 — Confirm numbering

1. List existing GXX files in `documents/guides/`
2. If the directory does not exist, create it first: `mkdir -p documents/guides`
3. Find the next available number (G01, G02 …)
4. **If a guide for the same service and operation type already exists**, ask the user whether to update the existing file rather than create a new one

---

## Step 3 — Gather information

Read the following content as the basis for writing:

- `CONTEXT.md` (if it exists) — to ensure consistent terminology
- Module docs in `documents/modules/` related to that service
- The project's `package.json`, `Makefile`, `README.md`, etc., to find existing commands

---

## Step 4 — Write the guide

Write following the structure below, **quick commands must come first**:

```markdown
# G0X — [Operation name]

## Quick commands

​```bash
# Commands ready to use directly
​```

---

## Prerequisites
(Fill in only when there are non-obvious prerequisites)

## Detailed steps
(Step-by-step instructions, including the command and a brief description for each step)

## Common issues
(List issues engineers actually encounter, not hypothetical ones)

## Related documents
(Associated FXX / RXX / BXX, if any)
```

**Writing principles:**
- Quick commands must be complete commands that can be copied and executed directly
- Detailed steps do not repeat what the quick commands already explain — only supplement the "why" and "edge cases"
- Avoid recording complete descriptions of every parameter — that is what `--help` is for
- Prerequisites only list non-obvious items (e.g., no need to write "requires Node.js to be installed")

---

## Step 5 — Write to file

Write the guide to `documents/guides/G0X-[operation-name].md`.

File naming rules:
- `G01-run-tests.md`
- `G02-build.md`
- `G03-start-dev-server.md`

---

## Step 6 — Completion report

```
GXX Guide created:

  ✓ documents/guides/G0X-[operation-name].md

Quick command preview:
  [Display quick command block content]
```

If this operation previously lacked documentation and engineers may have spent time searching for it, add a note:
"Next time just check `documents/guides/` directly — no need to ask AI again."
