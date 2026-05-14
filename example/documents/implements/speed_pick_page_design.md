---
author: highsunday0630@gmail.com
date: 2026-05-14
title: Speed Pick Page — one-button pick workflow
uuid: 2d0b8a63a5f74b0a8d6e9c1f3b247901
version: 0.1.0
---

# Speed Pick Page Design

## 1. 功能概述 (Feature Overview)

Add a new **05 · Speed Pick** tab for operators who already trust a trained model and do not need the guided step-by-step flow in **04 · Pick**.

The existing Pick page is still the safer review workflow:

1. Select model
2. Observe / capture
3. Detect / inspect result
4. Execute
5. Done

The Speed Pick page compresses this into one screen:

1. Select a model on the left.
2. Select a pick TCP for that model on the right.
3. Press **Run**.
4. The robot runs the full pick workflow automatically.
5. The page shows the detected result image and whether detection quality is good.

This page should reuse the existing pick workflow API and model data instead of creating a separate backend pipeline.

---

## 2. User Story

- **As a** demo operator or production test user
- **I want** to choose a trained model, choose a pick TCP, and press one Run button
- **So that** the robot can complete observe, capture, detection, and pick execution without requiring the guided 04 Pick stepper

---

## 3. Scope

### In Scope

- Add a new top navigation tab: `05 · Speed Pick`.
- Add a new frontend route/page, proposed name: `speed-pick`.
- Use the same pickable model list as 04 Pick.
- Left side: model selection list similar to `PickScreen` select step.
- Right side: selected model information, pick TCP selector, result/status area, and Run button.
- Pressing Run starts the full robot workflow.
- Show live job state and logs while running.
- Show the detected result image after the workflow finishes or after detection result is available.
- Show detection quality clearly: `GOOD` / `BAD`, `detected`, `matches`, `inliers`, and `inlier_ratio`.
- Disable model/TCP selection and Run while a job is active.
- Lock page navigation while a Speed Pick job is active, so the user cannot switch to another tab and start another workflow.
- Demo Mode support using the existing mock API path.

### Out of Scope

- Removing or replacing 04 Pick.
- Manual review/confirm before motion in Speed Pick.
- Editing or teaching TCPs from Speed Pick.
- Creating new models from Speed Pick.
- New robot emergency stop API.
- New backend pose-estimation algorithm.
- Frontend TCP overlays on top of the result image.

---

## 4. Proposed Page Layout

The page should use a two-column workcell layout with a fixed visual priority:

- Left side: about 1/4 width for model selection.
- Right side: about 3/4 width for selected model info, TCP selection, Run, result image, quality status, and compact log.
- The operator must not need to scroll down on a normal desktop viewport to see the detection result area.
- The page can allow the left model list to scroll internally when many models exist.

### Left Column: Select Model

Reuse the visual behavior from 04 Pick:

- Panel title: `Choose model`
- List only pickable models from `fetchPickableModels()`.
- Each model row shows:
  - model name
  - point count
  - valid point count
  - pick pose count
- Selecting a model updates the right side immediately.
- Default selection:
  - If launched with an initial model payload, select that model if it is pickable.
  - Otherwise select the first pickable model.
- Empty state:
  - `No pickable model found. Build a model with pick pose first.`

### Right Column: Workcell Panel

Panel title: `Speed Pick Setup`

The right side should be one primary workcell panel instead of separate stacked setup/result panels. The panel should keep the workflow controls and result in the first viewport:

Top section:

- model name
- model ID
- point count and valid point count
- pick pose count
- first available object capture image, when available
- selected TCP pose name
- `Pick TCP` selector
- `Run` button
- optional `Open in 04 Pick` action

Controls:

- `Pick TCP` selector
  - Uses selected model `pick_poses`.
  - Defaults to the model's first pick pose.
  - Disabled if only one TCP exists or while running.
- `Run` button
  - Starts the full workflow.
  - Disabled until a model and TCP are selected.
  - Disabled while a job is active.
- Optional secondary action: `Open in 04 Pick`
  - Navigates to the guided Pick page for the same model.
  - Disabled while a Speed Pick job is active.

### Result Area

Show directly under the setup controls inside the same right workcell panel:

- Current state:
  - `READY`
  - `RUNNING`
  - `DETECTED`
  - `PICKED`
  - `FAILED`
- Detection result image:
  - Use `result_image_url` as a plain `<img>`.
  - No SVG, canvas, TCP marker, label, or overlay on the result image.
- Detection quality:
  - `GOOD` should show a positive status.
  - `BAD` or `detected=false` should show a failure status.
- Metrics:
  - matches
  - inliers
  - inlier ratio
  - reason
  - estimated pick pose, when available
- RTDE / workflow logs:
  - Use the existing `TerminalLog` component.
  - Keep log deduplication behavior consistent with 04 Pick.
  - Keep this compact and lower priority than the result image so the result remains visible without page scrolling.

---

## 5. Workflow Behavior

### 5.1 Run Button

When the user presses **Run**, Speed Pick should call:

```ts
startPickWorkflow({
  model_id: modelId,
  pick_name: pickName,
  observe_name: 'observe',
  approach_height_m: 0.05,
  dry_run: false,
});
```

`dry_run=false` is required because Speed Pick is intended to run the complete process without a manual confirm step.

### 5.2 Job Polling

After `startPickWorkflow` returns `job_id`, poll:

```ts
pollJob(jobId)
```

Use the existing polling cadence from 04 Pick unless implementation finds a better shared helper.

Expected behavior:

- Append new log lines without duplicates.
- Update running state from `job.status` and `job.progress`.
- On `succeeded`, store `job.result` and show the final result.
- On `failed`, show `job.error.code` and `job.error.message`.
- Clear polling when the component unmounts or the job finishes.

### 5.3 Active Job Page Lock

While a Speed Pick workflow job is active, the user must not be able to leave the Speed Pick page through normal app navigation.

Locked actions:

- top navigation tabs, including Create, Viewer, Pick, and Speed Pick
- `Console home` / brand navigation
- `Open in 04 Pick`
- browser-style route changes inside the Electron shell, if any are added later

Required behavior:

- Navigation controls should be disabled or ignored while `activeJobId !== null`.
- The UI should show a visible running status so the user understands why navigation is locked.
- After the job reaches `succeeded` or `failed`, navigation becomes available again.
- If implementation centralizes routing in `AppShell`, Speed Pick should report its busy state upward so `TopBar` and route changes can be guarded globally.

### 5.4 Result Handling

The backend `PickWorkflowResult` already contains:

- `detected`
- `quality`
- `reason`
- `matches`
- `inliers`
- `inlier_ratio`
- `estimated_pick_pose`
- `captured_image_url`
- `result_image_url`
- `pick_execution`

Speed Pick should show the result image and detection quality after the job completes.

If the backend returns `quality='BAD'` or `detected=false`, the UI must not present the run as a successful pick, even if the job status is `succeeded`. The final status should distinguish:

- job succeeded and detection good: `PICKED`
- job succeeded but detection bad: `DETECTION BAD`
- job failed: `FAILED`

### 5.5 Selection State

When the selected model changes:

- Reset the selected TCP to that model's first pick pose.
- Clear the previous result and error.
- Clear `hasObservedPosition`-style state if shared code is used.

When selected TCP changes:

- Clear previous result and error.
- Do not start detection automatically.

---

## 6. Acceptance Criteria

### Scenario 1: Speed Pick tab is available

- **Given** the frontend app is open
- **When** the user looks at the top navigation
- **Then** a `05 · Speed Pick` tab is visible after `04 · Pick`
- **And** clicking it opens the Speed Pick page

### Scenario 2: Pickable models load on the left

- **Given** the backend has at least one built model with executable pick poses
- **When** the user opens Speed Pick
- **Then** the left column lists the same pickable models as 04 Pick
- **And** the first pickable model is selected by default

### Scenario 3: Selected model info appears on the right

- **Given** a model is selected
- **When** the right column renders
- **Then** it shows model name, model ID, point counts, pick pose count, object image if available, and selected TCP
- **And** the right side uses the primary 3/4 width workspace area

### Scenario 4: TCP selector lists model pick poses

- **Given** the selected model has multiple pick poses
- **When** the user opens the TCP selector
- **Then** every `pick_poses[].name` is available
- **And** changing TCP updates `pickName` before Run

### Scenario 5: Run starts the full workflow

- **Given** a model and TCP are selected
- **When** the user presses Run
- **Then** the page calls `POST /api/pick/workflow`
- **And** the request body contains `dry_run=false`, selected `model_id`, selected `pick_name`, `observe_name='observe'`, and `approach_height_m=0.05`
- **And** model/TCP selection and Run are disabled while the job is active
- **And** top navigation and page-exit actions are disabled or ignored while the job is active

### Scenario 6: Result image and good detection are shown

- **Given** the workflow job succeeds with `detected=true` and `quality='GOOD'`
- **When** polling receives the final job result
- **Then** the page shows `result_image_url` as an image
- **And** the page shows a good detection status
- **And** the page shows matches, inliers, inlier ratio, reason, and estimated pick pose
- **And** the result image is in the same right workcell panel as model info, TCP selection, and Run, so the operator does not need to scroll below a separate setup area

### Scenario 7: Bad detection is not shown as success

- **Given** the workflow job succeeds but returns `detected=false` or `quality='BAD'`
- **When** polling receives the final job result
- **Then** the page shows the result image if available
- **And** the final page status is `DETECTION BAD`, not `PICKED`
- **And** the reason is visible

### Scenario 8: Failed job shows error and allows retry

- **Given** the workflow job fails
- **When** polling receives `status='failed'`
- **Then** the page shows `error.code` and `error.message`
- **And** Run becomes enabled again after the job stops
- **And** the user can retry with the same model/TCP

### Scenario 9: No pickable models

- **Given** there are no built pick-executable models
- **When** the user opens Speed Pick
- **Then** the model list shows an empty state
- **And** the Run button is disabled

### Scenario 10: Active Speed Pick job blocks page changes

- **Given** a Speed Pick workflow job is active
- **When** the user clicks another top navigation tab, the brand/home navigation, or `Open in 04 Pick`
- **Then** the app remains on the Speed Pick page
- **And** no other page workflow can be started
- **And** navigation becomes available again after the active job succeeds or fails

### Scenario 11: Primary layout is simple and no-scroll for result

- **Given** the user opens Speed Pick on a normal desktop viewport
- **When** the page renders a selected model
- **Then** the model selector occupies the left 1/4 rail
- **And** the right 3/4 workcell contains model info, TCP selector, Run, result image/status, and compact RTDE log
- **And** the detection result area is visible without scrolling past a separate setup panel

---

## 7. Test Scenarios

| ID | Scenario | Given | When | Then | Priority |
|---|---|---|---|---|---|
| TC1 | Navigation tab | App loaded | User views top nav | `05 · Speed Pick` appears and opens page | High |
| TC2 | Model list | Pickable models exist | Open Speed Pick | Models render in left column; first selected | High |
| TC3 | Initial model payload | Route payload is a pickable model ID | Open Speed Pick | That model is selected | Medium |
| TC4 | Model details | A model is selected | Right panel renders | Model ID, counts, pick count, selected TCP shown | High |
| TC5 | TCP selector | Model has 2+ TCPs | Change selector | `pick_name` sent later matches selected TCP | High |
| TC6 | Single TCP | Model has 1 TCP | Right panel renders | TCP selector disabled or read-only | Medium |
| TC7 | Run request | Model and TCP selected | Click Run | `startPickWorkflow` called with `dry_run=false` | High |
| TC8 | Running lock | Job active | Try interacting | model/TCP/Run disabled | High |
| TC9 | Good result | Job succeeds with `quality='GOOD'` | Polling completes | Result image and good status shown | High |
| TC10 | Bad result | Job succeeds with bad detection | Polling completes | Status says detection bad, not picked | High |
| TC11 | Failed job | Job returns failed | Polling completes | Error shown; Run enabled for retry | High |
| TC12 | Demo mode | `VITE_DEMO_MODE=1` | Open and run Speed Pick | Mock job completes with deterministic result | Medium |
| TC13 | Navigation lock | Speed Pick job active | Click top nav / home / Open in 04 Pick | Route stays on Speed Pick; no other workflow starts | High |
| TC14 | Workcell layout | Page loaded on desktop viewport | Render Speed Pick | Left model rail uses 1/4 width; right workcell uses 3/4 width and contains setup, Run, result, and compact log | High |

---

## 8. Likely Affected Files

| File | Expected Change |
|---|---|
| `app/src/renderer/src/components/TopBar.tsx` | Add `speed-pick` route label as `05 · Speed Pick`; support disabled navigation while a job is active |
| `app/src/renderer/src/components/AppShell.tsx` | Add route handling for the new page; guard route changes while Speed Pick reports an active job |
| `app/src/renderer/src/screens/HomeScreen.tsx` | Optionally add a Speed Pick entry card or link |
| `app/src/renderer/src/screens/SpeedPickScreen.tsx` | New page implementing the one-screen workflow |
| `app/src/renderer/src/lib/apiTypes.ts` | Usually no change; reuse existing pick workflow types |
| `app/src/renderer/src/lib/api.ts` | Usually no change; reuse existing API facade |
| `app/src/renderer/src/lib/mockApi.ts` | Usually no change if `startPickWorkflow` mock already supports execute jobs |
| `app/src/renderer/src/screens/__tests__/SpeedPickScreen.test.tsx` | New tests for TC2-TC11 |
| `app/src/renderer/src/components/__tests__/TopBar.test.tsx` or route tests | Add navigation coverage if a test file exists |

Implementation should avoid duplicating large parts of `PickScreen.tsx`. If practical, extract shared helpers/components only when it reduces real duplication:

- model list rendering
- selected model preview
- pick workflow polling/log handling
- result quality panel

---

## 9. Design Notes

- Speed Pick is intentionally less interactive than 04 Pick. It should be visually clear that pressing Run can move the real robot.
- While Speed Pick is running, route changes should be blocked to avoid the user starting another page's Run action while the robot workflow is still active.
- The primary status should be based on both job status and detection quality.
- `dry_run=false` currently re-runs observe/capture/detect before physical execution. Speed Pick should rely on that backend behavior.
- If future safety requirements need a confirmation gate, that belongs in 04 Pick or a separate setting, not in the default Speed Pick flow.
- Do not overlay TCP markers on the result image. F05 already decided result images should remain backend-provided images without frontend overlays.

---

## 10. Documentation Follow-Up

After implementation, update:

- `documents/modules/GUI設計規劃.md`
  - Add Speed Pick to the route/page list.
  - Document the one-button workflow and relationship to 04 Pick.
- `documents/modules/frontend_demo_mode.md`
  - Add Speed Pick demo behavior and test notes if Demo Mode support is implemented.
- This design doc
  - Add implementation record, changed files, tests run, and known limitations.
