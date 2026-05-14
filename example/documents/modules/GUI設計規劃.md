---
author: highsunday0630@gmail.com
date: 2026-05-13
title: GUI 設計規劃
version: 0.2.0
---

# GUI 設計規劃

## 1. Purpose

This document describes the current GUI module for the SIFT 3D pose-estimation demo application.

The GUI lets an operator or UI reviewer complete the main demo workflows without using CLI commands:

1. Create a new 3D SIFT model from camera captures.
2. Inspect built models, captures, point clouds, build metadata, and pick poses.
3. Select a model and run the pick workflow with observe, detect, re-detect, execute, and done steps.

The application is intended for demo, research, validation, and UI review. It is not documented here as a production HMI.

## 2. Current Implementation Status

The current app is an Electron/Vite/React frontend with a Python Flask backend API.

The renderer has two operating modes:

- Real Mode: React screens call the backend at `http://localhost:3001`.
- Demo Mode: React screens use deterministic frontend-only mock APIs and do not require backend, robot, camera, or model files.

The frontend API boundary is `app/src/renderer/src/lib/api.ts`. Screens import API helpers from this facade instead of directly importing real or mock implementations.

## 3. Architecture

```text
Electron / React Renderer
  |
  v
app/src/renderer/src/lib/api.ts
  |
  +-- Real Mode -> realApi.ts -> Flask backend API -> filesystem / SIFT scripts / UR robot / Orbbec camera
  |
  +-- Demo Mode -> mockApi.ts -> deterministic in-memory and fixture-backed mock data
```

Important files:

| Area | Files |
| --- | --- |
| App shell and routing | `app/src/renderer/src/components/AppShell.tsx` |
| Top status bar | `app/src/renderer/src/components/TopBar.tsx` |
| Bottom status bar | `app/src/renderer/src/components/BottomBar.tsx` |
| API facade | `app/src/renderer/src/lib/api.ts` |
| Real backend API client | `app/src/renderer/src/lib/realApi.ts` |
| Demo mode detection and asset URL helper | `app/src/renderer/src/lib/demoMode.ts` |
| Mock API and data | `app/src/renderer/src/lib/mockApi.ts`, `app/src/renderer/src/lib/mockData.ts` |
| Create workflow | `app/src/renderer/src/screens/CreateScreen.tsx` |
| Viewer workflow | `app/src/renderer/src/screens/ViewerScreen.tsx` |
| Pick workflow | `app/src/renderer/src/screens/PickScreen.tsx` |
| Speed Pick workflow | `app/src/renderer/src/screens/SpeedPickScreen.tsx` |
| Shared point-cloud visualization | `app/src/renderer/src/components/viz/PointCloud3D.tsx` |
| Backend API entrypoint | `backend/app.py` |
| Pick backend workflow service | `backend/services/pick_workflow.py` |

## 4. UI Navigation

The shell has five primary routes:

| Route | Screen | Responsibility |
| --- | --- | --- |
| Home | `HomeScreen` | Entry point for Create, Viewer, and Pick workflows |
| Create | `CreateScreen` | Build a model and record pick poses |
| Viewer | `ViewerScreen` | Inspect existing built models |
| Pick | `PickScreen` | Guided pick workflow for a built model |
| Speed Pick | `SpeedPickScreen` | One-screen model/TCP selection and one-button full pick execution |

`AppShell` owns the active route and passes optional model IDs between screens. For example, Create can exit into Viewer or Pick with the newly created model ID. Speed Pick can report a busy state to `AppShell`; while busy, top navigation route changes are disabled to prevent starting another workflow during an active robot job.

## 5. Create Model Workflow

`CreateScreen` is a five-step workflow:

1. Setup
2. Observe
3. Build
4. Teach pick
5. Review

Current behavior:

- Default model names use `model_YYYYMMDD_HHMM`.
- Coordinate frame and unit are shown but disabled in the current UI.
- If model creation reports that a model already exists, the UI can expose `Force overwrite existing model`.
- Observe captures three images and allows ROI painting on image 1.
- Image 2 and image 3 are display-only thumbnails.
- Users may continue without drawing an ROI mask; the full image is used.
- `Re-capture images` resets old captures and mask state.
- Continuing from Observe starts the build job and moves to Build.
- Build displays triangulation status, point-cloud growth, and pipeline logs.
- Build completion enables continuing to Teach pick.
- Teach pick records one or more pick TCP poses and allows deleting extra pick rows.
- Review summarizes build metadata and recorded pick poses, then lets the user open Viewer or Pick.

Main backend/API helpers:

- `createModel`
- `startCapture`
- `pollJob`
- `fetchModelCaptures`
- `uploadMask`
- `startBuild`
- `fetchModel3dView`
- `fetchModelDetail`
- `savePickPose`

## 6. Viewer Workflow

`ViewerScreen` is the model inspection page.

It includes:

- model list
- selected model header
- Geometry tab
- Images tab
- Pick poses tab
- Raw JSON tab

Current behavior:

- The header emphasizes model quality with valid percentage and valid point count.
- Coordinate frame is still available in detail metadata but is no longer the primary status pill.
- Geometry tab shows the sparse point cloud and object image together.
- Capture images are fetched when a model is selected, so object image preview is available without switching to Images.
- Point-cloud UI is simplified: auto-rotate remains visible, while frusta and axes are hidden in the Viewer geometry tab.
- Pick poses are inspect-only in Viewer; adding pick poses belongs to the Create workflow.
- The model library includes a destructive `Delete model` action for the selected model. The action opens a warning dialog first; confirmation calls the backend delete API and removes the model from the list only after the backend succeeds.

Main backend/API helpers:

- `fetchViewerModels`
- `fetchModel3dView`
- `fetchModelCaptures`
- `deleteModel`

## 7. Pick Workflow

`PickScreen` guides the pick demo sequence.

Main steps:

1. Select model
2. Observe
3. Detect
4. Execute
5. Done

Current behavior:

- Select step shows pickable models and a selected-model preview.
- The selected model panel displays the first available object capture image.
- Unused motion controls were removed from the selection step.
- Pick pose selection is handled in the Detect step, including multi-TCP models.
- `Continue -> Observe` starts the dry-run observe workflow.
- After the first observe move succeeds, the screen tracks that the robot is already at observe position.
- `Re-capture image` can capture from the current observe position without commanding another observe move.
- Re-detect can use a different pick pose and remains on the Detect step after completion.
- Execute describes the current motion sequence: approach, pick TCP, dwell, then return to observe.

Backend support:

- Pick workflow requests include optional `skip_observe_move`.
- When `skip_observe_move` is true, the backend reads the current TCP pose instead of moving to the configured observe pose.
- Execution command generation includes `--pick-dwell-s 3.000000`.

Main backend/API helpers:

- `fetchPickableModels`
- `fetchModelCaptures`
- `startPickWorkflow`
- `pollJob`

## 8. Speed Pick Workflow

`SpeedPickScreen` provides a faster one-screen path for users who already trust a trained model and TCP.

Current behavior:

- Top navigation includes `05 · Speed Pick`.
- The left 1/4 model rail lists the same pickable models as the guided Pick page.
- The right 3/4 workcell panel shows selected model details, first available object capture image, pick pose count, TCP selector, Run button, detection result, and compact RTDE log in one primary view.
- Pressing Run calls `startPickWorkflow` with `dry_run=false`, `observe_name="observe"`, `approach_height_m=0.05`, selected `model_id`, and selected `pick_name`.
- While a Speed Pick job is active, model selection, TCP selection, Run, Open in 04 Pick, and top navigation are disabled or ignored.
- The detection result is kept in the right workcell instead of below a separate setup stack, so operators do not need to scroll down to inspect the result on a normal desktop viewport.
- The result area shows the backend result image as a plain image, without frontend TCP/SVG/canvas overlays.
- Final status distinguishes `PICKED`, `DETECTION BAD`, and `FAILED`.
- Demo Mode uses the same API facade and mock execute job path.

Main backend/API helpers:

- `fetchPickableModels`
- `fetchModelCaptures`
- `startPickWorkflow`
- `pollJob`

Related document:

- `documents/implements/F09-SpeedPickPage.md`

## 9. Demo Mode

Demo Mode exists for UI design review and frontend testing without hardware.

Run Demo Mode from the Electron app directory:

```bash
cd app
VITE_DEMO_MODE=1 npm run dev
```

Run Real Mode by omitting the environment variable:

```bash
cd app
npm run dev
```

Current behavior:

- `isDemoMode()` checks `import.meta.env.VITE_DEMO_MODE === "1"`.
- `api.ts` dispatches API calls to `mockApi` or `realApi`.
- `resolveAssetUrl()` keeps image URLs mode-safe.
- Top and bottom bars show visible Demo Mode status.
- Demo Mode provides deterministic models, images, point-cloud data, logs, and async job simulation.

Related document:

- `documents/modules/frontend_demo_mode.md`

## 10. Shared UI Components

Important shared UI pieces:

- `Panel`: repeated container component for workflow sections.
- `Pill`: compact status labels with `dim`, `ok`, `warn`, `bad`, `info`, and `purple` variants.
- `TerminalLog`: log display with optional fixed height and auto-scroll.
- `PointCloud3D`: SVG-based point-cloud visualization with drag rotation, right-drag pan, and scroll zoom.
- `PoseAxisOverlay`: pose/result overlay used by pick result views.

Global styling lives in:

- `app/src/renderer/src/globals.css`

Prototype/reference styling also exists under:

- `ui_design/`

## 11. Data and State Flow

The main frontend data pattern is:

1. Screen starts a backend or mock job.
2. Backend/mock API returns a `job_id`.
3. Screen polls `pollJob(job_id)`.
4. Logs are appended incrementally.
5. On success, the screen fetches final data such as captures, model detail, point cloud, or pick result.
6. UI state advances only when required data is ready.

State is local to each screen. There is no global state manager.

## 12. Testing Notes

Frontend tests:

```bash
cd app
npm test
```

Backend tests:

```bash
pytest backend/tests
```

Current tested UI areas include:

- Demo Mode API facade and URL handling
- Create model setup, capture, mask, build, teach, and navigation behavior
- Viewer model list, geometry, captures, pick poses, and error states
- Pick model selection, observe/detect, re-detect, recapture, TCP selection, and execution flow
- Speed Pick navigation, model/TCP selection, one-button execution, route lock, result states, and Demo Mode execution
- Backend pick workflow request parsing and command generation

## 13. Constraints and Known Limitations

- The GUI remains a demo/research tool, not a hardened production HMI.
- Real Mode depends on the backend, filesystem layout, SIFT scripts, UR robot, and Orbbec camera availability.
- Demo Mode is deterministic and useful for review, but it does not simulate exact robot kinematics or camera hardware behavior.
- Speed Pick intentionally skips the guided Pick confirmation step; use 04 Pick when manual review is needed before execution.
- Create workflow currently keeps frame/unit fixed in the UI.
- Viewer is inspect-only for pick poses; pick-pose editing is handled in Create.
- UI state is local to screens, so cross-screen persistence depends on backend or mock API state.

## 14. Related Documents

- `documents/uiux_improve.md`
- `documents/modules/frontend_demo_mode.md`
- `documents/modules/backend-api.md`
- `documents/modules/backend-camera-manager.md`
- `documents/implements/F01-PickPage.md`
- `documents/implements/F02-ViewerPage.md`
- `documents/implements/F03-CreatePage.md`
- `documents/implements/F04-CreatePage-ObserveUI.md`
- `documents/implements/F05-PickPage-DetectTcpSelector.md`
- `documents/implements/F06-Frontend-DemoMode.md`
- `documents/implements/F07-Frontend-DemoMode-RealFixtureData.md`
- `documents/implements/F09-SpeedPickPage.md`
