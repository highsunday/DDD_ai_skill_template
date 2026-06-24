---
name: ddd-import-example
description: Reference the minimal runnable example exported from another project by ddd-export-example, located under reference-examples/import/, to implement a similar feature in the current project. Understand the example and MANIFEST, align with the current project context (same-stack uses code directly; cross-stack translates core concepts from MANIFEST as best as possible), generate an FXX draft that references the example, and convert the example smoke test into acceptance criteria. Hand off to ddd-tdd after human review. Use when the user has already copied another project's example bundle into reference-examples/import/ and wants to add a similar feature to this project.
---

# DDD Import Example

Reference a minimal runnable example exported from another project to implement a similar feature in the current project. The export side is `ddd-export-example`; this skill does not fetch files across repos itself — the example must be **manually copied** by the user into `reference-examples/import/` before this skill runs.

## Position in the DDD pipeline

This skill **does not modify code directly**. It transforms an "external example" into an "FXX document for this project", then hands back to the existing DDD flow:

```
Another project's export bundle  →（user manually copies）→ reference-examples/import/<feature>/
        ↓ ddd-import-example
Understand example + align with current project context
        ↓
Generate FXX draft referencing the example (with acceptance criteria = translated smoke test)
        ↓ human review
ddd-tdd red → green → acceptance
```

Tenet: **No ambiguous document enters ddd-tdd.** No matter how good the example is, it must first be distilled into an FXX in the current project's context.

---

## Step 1 — Locate the example bundle

1. List the bundles under `reference-examples/import/`.
2. If the directory does not exist or is empty: remind the user "Please first use ddd-export-example to export from the source project, then copy that bundle folder into this project's `reference-examples/import/`", then stop.
3. If there are multiple bundles, ask the user which one to reference this time.

---

## Step 2 — Understand the example

Read in order:

1. **`MANIFEST.md`** (read first) — feature summary, entry point, file map, core concepts, rewrite guide, smoke test summary.
2. **Entry file → core source files per the file map** — understand the essential flow.
3. **Smoke test** — understand what "runnable" concretely verifies.

Summarize to the user in a few sentences "I understand what this example does", and confirm there is no misunderstanding before proceeding.

---

## Step 3 — Align with current project context

Read project information to align:

- `CONTEXT.md` (if it exists) — terminology, architectural boundaries.
- Relevant module documents under `documents/modules/` — existing responsibility boundaries.
- The project's dependency list / existing similar code — confirm the tech stack and conventions.

Determine same-stack or cross-stack:

- **Same-stack (same language / framework)**: Use the example's **concrete code** as the primary reference, mapping each section to the current project's structure and naming.
- **Cross-stack (different language)**: Use the **"Core Concepts (language-agnostic)" section** of the MANIFEST as the primary reference, and **translate as best as possible** to rebuild on the current project's stack based on the essential flow; treat the concrete code as supplementary only.

Compile what to keep and what to replace against the MANIFEST's "rewrite guide":

```
Alignment plan (feature: <feature>, same-stack / cross-stack):

Keep (functional essence):
  - <core logic>

Replace with current project context:
  - <example's fake DB> → this project's <real data layer>
  - <example naming> → corresponding terms from this project's CONTEXT.md

Needs user decision:
  - <ambiguous or missing information>
```

---

## Step 4 — Generate FXX draft

1. Find the next available number under `documents/implements/` (F01, F02, …; if the directory does not exist, `mkdir -p documents/implements`).
2. Write the FXX following this project's F00 template (`documents/implements/F00-feature-template.md`, if available), focusing on:
   - **Feature overview**: Describe the feature and note "Reference example: `reference-examples/import/<feature>/`".
   - **Requirements**: Write a User Story in the current project's user-role language.
   - **Acceptance criteria / test scenarios**: Translate the **smoke test summary** from the example's MANIFEST into **Given-When-Then in the current project's context** as at least one acceptance criterion. The path that made the example "runnable" is the path this project must turn green first.
   - **Implementation notes**: Attach the alignment plan — what to carry over, what to replace, translation focus points for cross-stack — and flag the "common pitfalls" from the MANIFEST.
3. Write to `documents/implements/F0X-<feature>.md`.

---

## Step 5 — Hand back for human review and ddd-tdd

Report and stop at the review checkpoint:

```
FXX draft generated (reference example: reference-examples/import/<feature>/):

  ✓ documents/implements/F0X-<feature>.md

Alignment approach: <same-stack, referencing code directly / cross-stack, translating from core concepts>
Acceptance criteria source: example smoke test → translated to this project's Given-When-Then

Next steps:
  1. Please review this FXX (especially whether the acceptance criteria and "replace" list are correct).
  2. After review is approved, run /ddd-tdd to drive the red → green → acceptance implementation.
```

**Do not skip the review and implement directly.** After review is approved, `ddd-tdd` takes over.
