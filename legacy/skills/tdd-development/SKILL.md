# TDD Development Skill

Use this skill when implementing a new feature or changing production code for a feature.

This workflow is driven by an `FXX` feature requirement document, such as:

- `F00-功能需求書模板.md`

- `F01-...md`

- `F02-...md`

The `FXX` document is the source of truth for expected behavior, acceptance criteria, business rules, boundaries, and test cases.

## Core Rule

Never start by changing production code for a behavior change.

Follow this required sequence:

1. Read and understand the provided `FXX` feature requirement document.

2. Identify the target feature scope, expected behavior, acceptance criteria, documented test cases, business rules, and open questions from the `FXX` document.

3. Inspect the repository to identify:

   - language

   - framework

   - package manager

   - test runner

   - existing test style

   - related production code

   - related test files

4. Map the `FXX` test cases and acceptance criteria to concrete automated tests.

5. Write or update the smallest test that should fail for the right reason.

6. Run the smallest relevant test command and confirm the failure.

7. Implement the minimum production code needed to pass.

8. Run the same test again and confirm it passes.

9. Run related tests if the change could affect nearby behavior.

10. Verify whether the implementation successfully fulfills the original `FXX` document.

11. Update the `FXX` document with implementation status, test evidence, changed files, decisions made, and any uncovered or deferred items.

12. Summarize the work.

## FXX Requirement Document Integration

Every feature implementation task must be connected to an `FXX` feature requirement document.

Before coding, ask for or locate the relevant `FXX` document.

Do not treat the user's short instruction alone as the complete requirement when an `FXX` document is expected.

Use the `FXX` document to determine:

- feature goal

- user story or usage scenario

- business rules

- input conditions

- output expectations

- acceptance criteria

- documented test cases

- edge cases

- error handling

- permissions or role-based behavior

- dependencies

- non-goals

- testable behavior

If the `FXX` document contains ambiguity, missing rules, or conflicting requirements, identify them clearly before implementation and ask the user to clarify.

If the user explicitly asks to proceed despite ambiguity, continue with clearly stated assumptions and record those assumptions in the `FXX` document.

## Supported F Document Types

The user may provide three types of F documents.

When a document is provided, determine its role before implementation.

### 1\. F00 功能需求書模板

Example:

```text
doc/F00-功能需求書模板.md
```

Use this as the structure and standard for writing or updating feature requirement documents.

Follow this template when creating or completing an `FXX` document.

### 2\. FXX 功能需求文件

Example:

```text
doc/F01-會員登入功能.md
doc/F02-訂單查詢功能.md
```

Use this as the source of truth for the actual feature implementation.

Map tests and implementation back to this document.

### 3\. FXX 實作或檢驗文件

If the provided F document includes implementation notes, test records, verification checklists, or completion sections, update those sections after implementation.

If such sections do not exist, add a clearly named section for implementation records.

## Before Coding

First inspect the repository.

Identify:

- language

- framework

- package manager

- test runner

- test command

- existing test structure

- existing tests near the target code

- related production files

- relevant `FXX` document path

- documented test cases in the `FXX` document

Prefer matching existing test patterns over inventing new ones.

If no test infrastructure exists, ask before adding a new framework.

If no `FXX` document is provided, ask the user to provide one.

If the user wants to create a new feature but only provides an informal description, help convert it into an `FXX` feature requirement document before coding.

## FXX Test Case Mapping

Before writing automated tests, extract the test cases already documented in the `FXX` file.

For each relevant documented test case, identify:

- the related acceptance criterion or business rule

- the behavior under test

- required input conditions

- expected output or state change

- edge case or error case coverage

- the automated test file and test name to create or update

Do not create a separate behavior-comment preflight when the `FXX` document already contains detailed test cases.

Use the repository's existing test naming convention. If no convention is clear, use a descriptive behavior-level test name such as:

```text
test_should_<expected_behavior>_when_<condition>
```

## Red Phase

Before implementation:

1. Add or update the smallest automated test derived from the `FXX` documented test cases.

2. Ensure the test maps to one or more acceptance criteria, business rules, or test cases in the `FXX` document.

3. Run only the targeted test first.

4. Confirm the test fails for the expected reason.

5. If the test does not fail, explain why and adjust it.

Do not skip the failing-test step unless the user explicitly says to skip TDD.

Do not weaken existing tests to make the new test pass.

Do not mock away the behavior being tested.

## Green Phase

After the test fails correctly:

1. Make the smallest production-code change needed to pass.

2. Avoid broad rewrites.

3. Avoid unrelated cleanup.

4. Do not change public APIs unless required by the `FXX` document.

5. Run the targeted test until it passes.

The production code must implement the behavior described in the `FXX` document, not only satisfy the test superficially.

## FXX Verification Phase

After implementation and passing tests, verify the result against the original `FXX` document.

Check:

- whether each relevant documented test case is covered by automated tests

- whether each acceptance criterion is covered by tests

- whether each business rule is implemented

- whether edge cases are handled

- whether error cases are tested

- whether any requirement was skipped, changed, or deferred

- whether any assumption was introduced during implementation

- whether implementation behavior differs from the original requirement

If the implementation does not fully satisfy the `FXX` document, clearly report the gap and do not mark the feature as complete.

## FXX Document Update Requirement

After implementation, update the related `FXX` document.

The update should include a section such as:

````markdown
## Implementation Record

### Status

Implemented / Partially Implemented / Not Implemented

### Implementation Summary

Describe what was implemented.

### Test Coverage

List the tests added or updated, including which FXX test cases they cover.

### Files Changed

#### Production Files

- ...

#### Test Files

- ...

### Acceptance Criteria Verification

| Acceptance Criterion | Status | Evidence |
| --- | --- | --- |
| ... | Passed / Partial / Failed | Test name or explanation |

### FXX Test Case Verification

| FXX Test Case | Status | Automated Test Evidence |
| --- | --- | --- |
| ... | Passed / Partial / Failed | Test name or explanation |

### Commands Run

```bash
...
```

### Result

Describe failing and passing test results.

### Assumptions

List any assumptions made during implementation.

### Deferred Items

List anything not implemented and why.

### Notes

Add risks, follow-up suggestions, or known limitations.
````

If the existing `FXX` document already has a similar section, update the existing section instead of creating a duplicate.

Do not remove original requirements unless the user explicitly asks to revise the requirement.

If implementation reveals that the requirement was inaccurate or incomplete, add notes instead of silently changing the requirement.

## Safety Rules

Do not claim success without running tests, unless the environment makes testing impossible.

If tests cannot be run, clearly say why.

Do not delete or weaken tests to make code pass.

Do not mock away the behavior being tested.

Prefer behavior-level tests over implementation-detail tests.

Do not modify production code before reading the `FXX` document and adding or updating the failing test.

Do not skip updating the `FXX` document after implementation.

Do not mark the `FXX` requirement as complete unless verification confirms that it is covered by implementation and tests.

## Final Output Format

When done, respond with:

````markdown
## Changed

Production files changed:

- ...

Test files changed:

- ...

FXX documents updated:

- ...

## TDD Evidence

Failing test observed:

- ...

Passing test observed:

- ...

Commands run:

```bash
...
```

## FXX Verification

Requirement document:

- ...

FXX test cases verified:

- ...

Acceptance criteria verified:

- ...

Implementation status:

- Implemented / Partially Implemented / Not Implemented

Gaps or deferred items:

- ...

## Notes

Any risks, skipped tests, assumptions, or follow-up suggestions.
````