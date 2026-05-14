# Doc Maintain

This skill manages engineering documents in `documents/` and keeps them aligned with the current implementation.

## Document Structure

Project documents live under:

- `documents/implements/`

   - `FXX-*`: feature requirement documents

   - `RXX-*`: refactor task documents

   - `BXX-*`: bug-fix documents

   - Templates:

      - `F00-功能需求書模板.md`

      - `R00-重構任務模板.md`

      - `B00-Bug修正模板 .md`

- `documents/modules/`

   - High-level module documentation for engineers.

   - These documents explain architecture, responsibilities, data flow, important files, boundaries, and known limitations.

   - They should avoid excessive line-by-line implementation detail.

## Core Goal

Keep documents and implementation consistent.

When documents and code disagree, prefer the current implementation as the source of truth unless the user explicitly says the document describes intended future behavior.

## Creating FXX / RXX / BXX Documents

1. Identify the document type:

   - New feature: `FXX`

   - Refactor: `RXX`

   - Bug fix: `BXX`

2. Inspect existing documents:

   - List `documents/implements/`

   - Find the next available number for the selected prefix.

   - Read the matching template.

   - Read related FXX/RXX/BXX documents if they overlap.

3. Inspect module context before writing:

   - Search `documents/modules/` for related feature names, components, routes, services, or concepts.

   - Read relevant module docs.

   - Search the codebase with `rg` for related files and implementation terms.

   - Use module docs to understand high-level architecture, but verify important claims against source code.

4. Write the new document:

   - Follow the matching template structure.

   - Make acceptance criteria testable.

   - Include concrete test scenarios.

   - Include likely affected modules and files.

   - Clearly mark assumptions, open questions, and out-of-scope items.

   - Do not over-specify implementation details unless required for correctness or compatibility.

5. Before finishing:

   - Check whether the new requirement implies a new module document or an update to an existing one.

   - Tell the user what module docs should be created or updated.

## Auditing Module Documents

1. Read relevant files in `documents/modules/`.

2. Search the implementation for every important claim:

   - framework/library versions

   - routes/pages

   - components

   - data sources

   - state flow

   - API/service/database behavior

   - tests

   - known limitations

3. Classify mismatches:

   - stale path or file structure

   - missing module or behavior

   - behavior changed in code but not document

   - document describes future intent as current implementation

   - too much low-level detail that will become stale quickly

4. Update module docs to reflect current code:

   - Keep them high-level and engineer-oriented.

   - Include file paths as navigation anchors.

   - Explain responsibilities and boundaries.

   - Preserve useful historical or roadmap notes only if clearly labeled.

5. Mention any code areas that still lack module documentation.

## After Implementation Work

After completing code changes:

1. Re-open the related FXX/RXX/BXX document.

2. Update implementation notes with:

   - final behavior

   - changed files/modules

   - test coverage added or updated

   - tests run and results

   - known limitations or follow-up work

3. Check `documents/modules/`:

   - If an existing module changed meaningfully, update its document.

   - If a new module, feature area, service, data flow, or architectural boundary was introduced, suggest creating a new module document.

4. If the user asked for full documentation sync, make the module document update directly.
   Otherwise, provide a concrete recommendation.

## Module Document Writing Standard

A good module document should help a new engineer quickly understand and navigate the code.

Prefer sections like:

- Purpose

- Current implementation status

- Key files

- Main responsibilities

- Data/state flow

- External dependencies

- Important constraints

- Known limitations

- Testing notes

- Related FXX/RXX/BXX documents

Avoid:

- copying large code snippets

- documenting every prop or local variable

- describing behavior that is not implemented yet as if it exists

- duplicating details already obvious from code

- leaving stale paths or old folder names

## Verification Habits

Use `rg` or `rg --files` before slower search tools.

Before editing documents, inspect:

- `documents/implements/`

- `documents/modules/`

- related source files

- `package.json` when framework/library versions matter

- tests or test configuration when documenting verification

When uncertain, write uncertainty explicitly instead of inventing missing implementation details.