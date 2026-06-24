---
name: ddd-export-example
description: Distills a feature from the current project into a "minimum runnable example" and outputs it to reference-examples/export/, so other projects can reference the implementation via ddd-import-example. The user points to the feature entry point; AI traces the dependency graph, confirms boundaries first, then distills into a standalone runnable minimum (with its own scaffold, smoke test, and MANIFEST). The smoke test must actually pass green before the export is considered complete. Use this when the user wants to export a successfully working feature from the current project as an example for reproducing similar functionality in another project.
---

# DDD Export Example

Distills a **successfully working feature** from the current project into a **standalone minimum runnable example**, outputting it to `reference-examples/export/<feature>/`.

## Why this skill exists

As projects grow complex, having AI add features directly inside them becomes error-prone. But with a **verified-runnable minimum example** as a reference, AI's success rate for reproducing similar features in other projects improves dramatically. This skill is responsible for "extracting a successful feature into a clean, runnable, portable example"; the corresponding import side is `ddd-import-example`.

## Output definition

| Property | Description |
|------|------|
| **Standalone runnable** | Strips project noise, includes minimum scaffold, can be executed independently |
| **Green-light verified** | Contains a smoke test — **must actually pass green after distillation to be considered complete** |
| **Portable** | The entire folder can be manually copied by the user to another project's `reference-examples/import/` |
| **Same-stack primary, cross-stack best-effort** | Code retains original language; MANIFEST also includes a language-agnostic "core concepts" section so different stacks can reference it |

---

## Step 1 — Confirm feature entry point

Ask the user (if not specified in the command):

1. **Which feature to export?** (one-sentence description, e.g. "JWT login", "file upload to S3")
2. **Where is the entry point for this feature?** (file / route / class / function — at least one starting point)

Do not ask the user to manually list all files — the user only needs to provide the entry point; dependencies are traced by AI.

---

## Step 2 — Trace dependencies and confirm boundaries (critical — confirm before distilling)

1. Start from the entry point, trace import / require / call relationships, and build a **dependency graph**.
2. Classify into three categories:
   - **Core**: the essential logic that implements this feature — must be included.
   - **Boundary dependencies**: frameworks, databases, external services, etc. → usually **replaced with minimum stubs / fakes**, not pulled in wholesale.
   - **Irrelevant noise**: project configuration, other features unrelated to this one → **excluded**.
3. Compile the results into a list for the user to confirm:

```
Planned minimum set (feature: <feature>):

Core (distilled as-is):
  - path/to/core-a.ts
  - path/to/core-b.ts

Boundary dependencies (will be replaced with stub / fake):
  - Database access → in-memory fake
  - External API → mock response

Excluded (unrelated to this feature):
  - Other modules, project configuration…

Uncertain, needs your decision:
  - <list boundary-ambiguous items>
```

4. **Wait for the user to confirm / revise the boundaries before proceeding to distillation.** Wrong boundaries make the entire distillation worthless.

---

## Step 3 — Confirm output location and naming

1. Ensure the folder exists: `mkdir -p reference-examples/export`
2. Choose a kebab-case name from the feature as the bundle folder, e.g. `reference-examples/export/jwt-login/`
3. If a bundle with the same name already exists, ask the user whether to update or rename — **do not overwrite directly**.

---

## Step 4 — Distill into minimum runnable example

Generate the following under `reference-examples/export/<feature>/`:

```
reference-examples/export/<feature>/
├── MANIFEST.md          ← Example specification (see Step 5)
├── src/                 ← Distilled core code (original language)
├── <scaffold file>      ← Minimum dependency list and run configuration (see below)
└── <smoke test file>    ← Test proving it runs
```

**Distillation principles:**
- Keep only the essential logic of the feature. Remove parameters, configuration, and abstraction layers unrelated to the feature.
- Replace external dependencies (DB, network, filesystem, third-party SDKs) with **minimum stubs / fakes** so the example runs without a real environment.
- Preserve original naming and structural conventions so the reader can understand "what it originally looked like".
- **Scaffold must be minimal**: provide the minimum dependency list and a single-line run command appropriate for the language (e.g. Node's `package.json` + `npm test`; Python's `requirements.txt` + `pytest`).
- **Smoke test is proof of the green light**: must cover at least the single most critical happy path of this feature, runnable with a one-line command.

---

## Step 5 — Write MANIFEST.md

Read the `templates/MANIFEST-template.md` from this skill's folder, fill it in with the actual content, and write it into the bundle. The MANIFEST must include:

- **Feature summary**: what this example demonstrates.
- **Entry point**: which file / function to start reading from.
- **File map**: one line describing the responsibility of each file.
- **Dependencies and execution**: what is required, which one-line command to run it, how to run the smoke test.
- **Core concepts (language-agnostic)**: describe in plain text the essential flow and key decisions of this feature, **without binding to any language** — this is the primary reference for cross-stack imports.
- **Adaptation guide**: when importing into another project, which parts are skeleton to keep and which are points to replace with the target project's context.
- **Smoke test summary**: what this test verifies (the import side will translate it into the target project's acceptance criteria).

---

## Step 6 — Run green-light verification (required — export is not allowed if it fails)

1. Install minimum dependencies in the bundle directory and execute the smoke test.
2. **If red light**: fix the example (usually a missing dependency or insufficient stub), re-run, until green light.
3. **If unable to run in the current environment** (e.g. missing runtime), honestly tell the user "green light not yet verified" and explain what is missing — **do not falsely report completion**.

---

## Step 7 — Completion report

```
Feature example export complete:

  ✓ reference-examples/export/<feature>/

Bundle contents:
  - MANIFEST.md
  - src/… (distilled core)
  - <scaffold>
  - <smoke test>

Green-light verification:
  ✓ <command executed> — passed
  (or: ✗ not yet verified, reason: …)

Next steps:
  Copy the entire reference-examples/export/<feature>/ folder to the target project's
  reference-examples/import/ directory, then run /ddd-import-example in the target project.
```
