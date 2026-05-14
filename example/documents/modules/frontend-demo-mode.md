---
author: highsunday0630@gmail.com
date: 2026-05-13
title: Frontend Demo Mode - run UI without backend, robot, or camera
uuid: 7f34d6c18a2b4e90b5c1d783f92a46de
version: 0.1.0
---

# Feature Requirement - Frontend Demo Mode

## 1. Feature Overview

The current app is designed as a full Electron/React frontend connected to a Python backend. The backend then talks to the SIFT 3D scripts, robot, camera, and filesystem.

For UI design review, this is too heavy. A UI designer should be able to open the frontend and test the Create, Viewer, and Pick workflows without:

- starting the Python backend
- connecting to a UR robot
- connecting to an Orbbec camera
- preparing real model files under `output/models`

This feature adds a frontend-only Demo Mode. In Demo Mode, the renderer uses deterministic mock data and simulated async jobs while keeping the same public frontend API shape used by the screens.

Important distinction:

- Backend `dry_run` still requires the backend process and API calls.
- Frontend Demo Mode must not require the backend at all.

## 2. Requirement / User Story

- **As a** UI designer
- **I want** to run the app in a demo mode with mock models, images, point clouds, logs, and pick results
- **So that** I can review and adjust the frontend experience without depending on backend, robot, or camera availability

## 3. Scope

### In Scope

- Add a frontend-only mode switch.
- Provide mock implementations for the existing renderer API helpers.
- Simulate async job progress for capture, build, save pick pose, and pick workflow.
- Make Create, Viewer, and Pick pages usable in Demo Mode.
- Show a visible Demo Mode indicator in the shell.
- Prevent any network request to `http://localhost:3001` when Demo Mode is enabled.
- Keep real backend mode unchanged.

### Out of Scope

- Changing Python backend behavior.
- Replacing backend `dry_run`.
- Persisting mock models permanently across app restarts.
- Simulating exact robot kinematics or camera hardware behavior.
- Making generated mock data physically accurate.
- Packaging a public production demo build.

## 4. Activation

The first implementation should use an environment variable:

```bash
VITE_DEMO_MODE=1 npm run dev
```

### How to Run Demo Mode

Run the frontend from the Electron app directory:

```bash
cd app
VITE_DEMO_MODE=1 npm run dev
```

No Python backend, robot, or camera is required. In Demo Mode the shell should show `DEMO MODE`, robot/camera should show simulated status, and the footer should show `DEMO MODE - NO BACKEND`.

To run normal Real Mode, omit the environment variable:

```bash
cd app
npm run dev
```

Recommended optional follow-up:

```text
?demo=1
```

or localStorage:

```text
localStorage.setItem("sift3d.demoMode", "1")
```

The initial version should prefer the environment variable because it is explicit, simple to test, and avoids accidental activation in real robot usage.

## 5. Architecture

Screens should continue importing from `@/lib/api`. The mode switch should live below that boundary.

Recommended file structure:

```text
app/src/renderer/src/lib/
  api.ts          // public API facade used by screens
  realApi.ts      // current backend fetch implementation
  mockApi.ts      // frontend-only mock implementation
  demoMode.ts     // mode detection helpers
  mockData.ts     // deterministic mock models/results/assets
```

The public exports from `api.ts` should remain stable:

- `fetchPickableModels`
- `startPickWorkflow`
- `pollJob`
- `fetchViewerModels`
- `fetchModel3dView`
- `fetchModelCaptures`
- `createModel`
- `startCapture`
- `uploadMask`
- `startBuild`
- `savePickPose`
- `fetchModelDetail`
- shared TypeScript interfaces

Implementation principle:

```ts
const api = isDemoMode() ? mockApi : realApi;
```

Avoid spreading mode checks through screen components.

## 6. Functional Requirements

### FR1: Demo Mode Detection

The renderer must expose a helper such as:

```ts
export function isDemoMode(): boolean
```

The initial implementation should return true when:

```ts
import.meta.env.VITE_DEMO_MODE === "1"
```

### FR2: No Backend Network Requests

When Demo Mode is enabled, the app must not call:

```text
http://localhost:3001
```

This includes image URLs. Existing hardcoded `BASE_URL` usage in screen components should be moved behind an asset URL helper.

Recommended helper:

```ts
export function resolveAssetUrl(url: string | null | undefined): string | null
```

It should support:

- backend relative paths such as `/api/files?...`
- absolute `http://` or `https://` URLs
- `data:image/...` URLs
- local frontend asset URLs

### FR3: Mock Models

Demo Mode should provide at least 3 built models:

- one simple model with 1 pick pose
- one model with multiple pick poses
- one model with mixed valid/unchecked/invalid points

Mock models should satisfy both Viewer and Pick page filters:

- `status = built`
- `pick_executable = true` for Pick page models
- realistic `point_count`, `valid_point_count`, and `pick_pose_count`

Default saved model requirement:

- Demo Mode should use `demo_mode_data/saved_models/mycard` as the default saved model fixture.
- Viewer Page should show `mycard` as the initially selected/default saved model when Demo Mode starts.
- Pick Page and Speed Pick Page should include `mycard` in the pickable model list by default.
- The `mycard` fixture should be loaded from its existing model artifacts where possible:
  - `demo_mode_data/saved_models/mycard/model.json`
  - `demo_mode_data/saved_models/mycard/sift_3d_model.json`
  - `demo_mode_data/saved_models/mycard/triangulated_points.json`
  - `demo_mode_data/saved_models/mycard/point_cloud_view.png`
  - `demo_mode_data/saved_models/mycard/captures/image_1.png`
  - `demo_mode_data/saved_models/mycard/captures/image_2.png`
  - `demo_mode_data/saved_models/mycard/captures/image_3.png`

### FR4: Mock Viewer Data

Viewer page should work across all tabs:

- Point cloud tab displays deterministic `ModelCloudData`.
- Source images tab displays mock image data or frontend asset URLs.
- Pick poses tab displays mock pick pose summaries.
- Raw JSON tab displays mock build metadata.

The point cloud should be deterministic, not random on every render, so UI review screenshots are stable.

For the default `mycard` model, Viewer data should prefer checked-in fixture files under `demo_mode_data/saved_models/mycard` instead of synthetic SVG/data-only placeholders. If a fixture field cannot be mapped exactly to the current frontend type, the mock layer may normalize it while preserving deterministic values.

### FR5: Mock Create Workflow

Create page should allow the designer to walk through:

1. Setup
2. Observe / capture
3. ROI save
4. Build
5. Teach pick
6. Review

Expected behavior:

- `createModel(name)` returns a generated mock model id.
- `startCapture(modelId)` returns a mock job id.
- `pollJob(jobId)` emits progress and logs, then succeeds.
- `fetchModelCaptures(modelId)` returns 3 mock captures after capture succeeds.
- `uploadMask(...)` succeeds without network.
- `startBuild(modelId)` returns a mock job id.
- `fetchModel3dView(modelId)` returns deterministic mock point cloud data.
- `savePickPose(...)` marks the pick pose as saved in in-memory mock state.

Persistence across reload is optional for the first version.

Capture page realism requirement:

- The Create Page capture step should use the checked-in capture fixtures:
  - `demo_mode_data/capure/image_1.png`
  - `demo_mode_data/capure/image_2.png`
  - `demo_mode_data/capure/image_3.png`
- The Create Page build/review mock point features should use:
  - `demo_mode_data/capure/sift_3d_model.json`
- Note: the directory is currently named `capure` in the repository. Use that exact path unless the folder is renamed in a separate cleanup.

### FR6: Mock Pick And Speed Pick Workflows

Pick page should allow:

1. Select a mock model.
2. Run observe/detect.
3. Re-detect with a different TCP when available.
4. Execute the simulated pick.
5. Reach the done screen.

Expected behavior:

- `startPickWorkflow(... dry_run=true)` returns a mock job id.
- `pollJob(jobId)` emits logs and a successful `PickWorkflowResult`.
- `startPickWorkflow(... dry_run=false)` returns a separate mock execute job.
- execute job progress should drive the existing approach/pick/retract animation.
- result data should include realistic `matches`, `inliers`, `inlier_ratio`, pose arrays, and image URLs.

Speed Pick page should use the same API facade:

- `fetchPickableModels()` returns the same mock pickable models used by Pick.
- `fetchModelCaptures(modelId)` returns object images for the selected model.
- `startPickWorkflow(... dry_run=false)` returns a mock execute job.
- `pollJob(jobId)` emits logs and a successful `PickWorkflowResult`.
- The page should show deterministic result image/status/logs without calling the backend.
- Active-job model/TCP/navigation lock behavior should match Real Mode.

### FR7: Shell Status

The top and bottom bars should clearly show Demo Mode.

Suggested UI behavior:

- TopBar robot/camera indicators show `DEMO` or `SIM`.
- BottomBar status text shows `DEMO MODE - NO BACKEND`.
- Avoid showing `ALL SYSTEMS NORMAL` in Demo Mode, because that suggests real hardware is connected.

### FR8: Real Mode Compatibility

When Demo Mode is disabled:

- existing backend API behavior must remain unchanged
- existing tests for real API request bodies should still pass
- no mock data should leak into production behavior

## 7. Acceptance Criteria

### Scenario 1: App starts in Demo Mode without backend

- **Given** the Python backend is not running
- **When** the user starts the frontend with `VITE_DEMO_MODE=1 npm run dev`
- **Then** the app opens without network errors
- **And** the shell displays a visible Demo Mode indicator
- **And** no request is made to `http://localhost:3001`

### Scenario 2: Viewer page works with mock data

- **Given** Demo Mode is enabled
- **When** the user opens Viewer page
- **Then** the saved model list shows mock built models
- **And** `mycard` from `demo_mode_data/saved_models/mycard` is available and selected by default
- **And** the point cloud tab renders mock points
- **And** the source images tab renders mock images
- **And** pick poses and raw data tabs show mock data

### Scenario 3: Create workflow can be completed

- **Given** Demo Mode is enabled
- **When** the user creates a model, starts capture, saves ROI, builds, records a pick pose, and reviews
- **Then** every step can be completed without backend calls
- **And** capture step renders `demo_mode_data/capure/image_1.png`, `image_2.png`, and `image_3.png`
- **And** build/review point feature data comes from `demo_mode_data/capure/sift_3d_model.json`
- **And** logs/progress are shown during simulated jobs
- **And** the final review page shows mock build metadata

### Scenario 4: Pick workflow can be completed

- **Given** Demo Mode is enabled and mock pickable models exist
- **When** the user runs observe/detect and then execute
- **Then** the UI reaches the done step
- **And** detection metrics, pose output, trajectory progress, and final TCP are populated

### Scenario 4b: Speed Pick workflow can be completed

- **Given** Demo Mode is enabled and mock pickable models exist
- **When** the user opens Speed Pick and presses Run
- **Then** the mock execute job completes
- **And** the page shows result image, detection status, metrics, pose output, and logs
- **And** no backend request is made

### Scenario 5: Real mode still calls backend

- **Given** Demo Mode is disabled
- **When** the user opens Create, Viewer, or Pick pages
- **Then** the app uses the real backend API helpers
- **And** existing request body behavior remains unchanged

### Scenario 6: Asset URLs work in both modes

- **Given** Demo Mode is enabled
- **When** a mock result image or capture image is rendered
- **Then** the image source is a data URL or frontend asset URL
- **And** no `localhost:3001` prefix is added

- **Given** Demo Mode is disabled
- **When** a backend result image or capture image is rendered
- **Then** relative backend URLs are resolved against the backend base URL

## 8. Test Scenarios

| ID | Scenario | Given | When | Then | Priority |
|----|----------|-------|------|------|----------|
| TC1 | Demo mode detection | `VITE_DEMO_MODE=1` | `isDemoMode()` is called | returns true | High |
| TC2 | Real mode detection | no demo env var | `isDemoMode()` is called | returns false | High |
| TC3 | No backend fetch in Demo Mode | backend is offline, demo enabled | Viewer page loads | no failed localhost request; mock models render | High |
| TC4 | Viewer default saved model | demo enabled | open Viewer page | `mycard` from `demo_mode_data/saved_models/mycard` is present and selected by default | High |
| TC5 | Viewer mock point cloud | demo enabled | open Viewer cloud tab | `PointCloud3D` receives deterministic points loaded or normalized from `mycard` fixture data | High |
| TC6 | Viewer mock captures | demo enabled | open Viewer images tab | `mycard` capture images render without backend URL | Medium |
| TC7 | Create mock capture job | demo enabled | click Start Capture | mock job logs/progress appear and `demo_mode_data/capure/image_1.png` through `image_3.png` become available | High |
| TC8 | Create mock build job | demo enabled | click Start triangulation | mock job succeeds and review metadata/features are populated from `demo_mode_data/capure/sift_3d_model.json` | High |
| TC9 | Pick default saved model | demo enabled | open Pick page | `mycard` is available as a pickable model by default | High |
| TC10 | Pick mock dry-run | demo enabled | click Move to observe & capture | detect step shows mock pose result | High |
| TC11 | Pick mock execute | demo enabled | click Execute Pick Motion | execute animation progresses and done step appears | High |
| TC12 | Real API compatibility | demo disabled | run API unit tests | real request body tests still pass | High |
| TC13 | Asset URL resolver | demo enabled and disabled | resolve data URL, absolute URL, and `/api/files` URL | each resolves correctly for current mode | Medium |
| TC14 | Speed Pick mock execute | demo enabled | open Speed Pick and click Run | mock execute job completes and result image/status/logs render | Medium |

## 9. Likely Affected Files

| File | Expected Change |
|------|-----------------|
| `app/src/renderer/src/lib/api.ts` | Become stable facade or keep types and delegate to real/mock implementations |
| `app/src/renderer/src/lib/realApi.ts` | Move current fetch-based implementation here |
| `app/src/renderer/src/lib/mockApi.ts` | Add frontend-only mock implementation |
| `app/src/renderer/src/lib/mockData.ts` | Load or normalize demo fixtures for saved models, point cloud, capture, and result data |
| `app/src/renderer/src/lib/demoMode.ts` | Add Demo Mode detection and asset URL helpers |
| `app/src/renderer/src/screens/CreateScreen.tsx` | Remove hardcoded backend asset URL usage; otherwise keep API calls unchanged |
| `app/src/renderer/src/screens/ViewerScreen.tsx` | Remove hardcoded backend asset URL usage; otherwise keep API calls unchanged |
| `app/src/renderer/src/screens/PickScreen.tsx` | Remove hardcoded backend asset URL usage; otherwise keep API calls unchanged |
| `app/src/renderer/src/screens/SpeedPickScreen.tsx` | Use API facade and mode-safe asset URLs for one-button mock execute flow |
| `app/src/renderer/src/components/TopBar.tsx` | Display Demo Mode robot/camera status |
| `app/src/renderer/src/components/BottomBar.tsx` | Display Demo Mode footer status |
| `app/src/renderer/src/lib/__tests__/api.test.ts` | Keep real API tests; add facade/mock mode tests if practical |
| `app/src/renderer/src/screens/__tests__/*.test.tsx` | Add focused Demo Mode render/workflow coverage |
| `demo_mode_data/saved_models/mycard/**` | Default saved model fixture for Viewer and Pick in Demo Mode |
| `demo_mode_data/capure/image_*.png` | Create Page capture image fixtures in Demo Mode |
| `demo_mode_data/capure/sift_3d_model.json` | Create Page build/review 3D feature fixture in Demo Mode |

## 10. Implementation Notes

### 10.1 Keep API Contracts Stable

Screen components should not know whether data comes from backend or mock code. This keeps the UI workflow realistic and avoids duplicate UI logic.

Bad direction:

```tsx
if (isDemoMode()) {
  setTimeout(...)
} else {
  startPickWorkflow(...)
}
```

Preferred direction:

```tsx
const { job_id } = await startPickWorkflow(params);
```

The imported function decides whether it is real or mock.

### 10.2 Mock Job Store

`mockApi.ts` should maintain a small in-memory job store:

```ts
type MockJobRecord = {
  job: JobResult;
  polls: number;
  finalResult?: PickWorkflowResult;
};
```

`pollJob(jobId)` can increment `polls` and return:

- poll 1: `running`, progress 20
- poll 2: `running`, progress 60
- poll 3: `succeeded`, progress 100

This is enough to test loading states, logs, and step transitions.

### 10.3 Demo Fixture Assets

Prefer realistic checked-in demo assets over generated placeholder images:

- Viewer / Pick saved model default:
  - `demo_mode_data/saved_models/mycard`
- Create capture images:
  - `demo_mode_data/capure/image_1.png`
  - `demo_mode_data/capure/image_2.png`
  - `demo_mode_data/capure/image_3.png`
- Create build/review features:
  - `demo_mode_data/capure/sift_3d_model.json`

The mock API should expose these assets through frontend-safe URLs or imported assets. It must not resolve them through `http://localhost:3001`.

If direct file serving from `demo_mode_data` is not available in Vite/Electron renderer, copy or import these files into a renderer-accessible asset location as part of implementation while preserving the source fixture path in documentation/tests.

Generated SVG or canvas data URLs are still acceptable only as fallback data for secondary mock models or result images where no real fixture exists.

### 10.4 Asset Path Notes

- The repository currently uses `demo_mode_data/capure`, not `demo_mode_data/capture`.
- Use the existing `capure` path unless a dedicated path cleanup renames the folder and updates all references.
- Tests should assert that Demo Mode image URLs do not include `http://localhost:3001`.

### 10.5 Error Simulation

The first version does not need a visible UI for choosing error states. However, mock API functions should be written so tests can later simulate:

- failed capture job
- failed build job
- failed pick workflow
- empty model list

## 11. Risks And Constraints

- If hardcoded `http://localhost:3001` remains in screen components, Demo Mode will still leak backend assumptions.
- If mock data does not match real TypeScript interfaces, the UI can pass demo review but break in real mode.
- If mock jobs complete immediately, designers cannot review loading/progress states.
- If Demo Mode is hidden, operators may confuse simulated hardware status with real robot readiness.

## 12. Documentation Follow-up

After implementation, update this document with:

- final changed files
- final activation command
- test coverage added
- tests run and results
- known limitations

Also consider updating `documents/modules/GUI設計規劃.md` with a short architecture note:

- Real Mode: Renderer -> Python Backend -> Robot/Camera/SIFT scripts
- Demo Mode: Renderer -> Mock API -> deterministic local mock data
