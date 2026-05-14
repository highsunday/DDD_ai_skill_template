---
author: highsunday0630@gmail.com
date: 2026-05-13
title: Frontend Demo Mode - realistic checked-in fixture data
uuid: 94a1a6e31d344e69a9a3a88a7816d0e5
version: 0.1.0
---

# 功能需求書 - Frontend Demo Mode Real Fixture Data

## 1. 功能概述 (Feature Overview)

F06 已新增 Frontend Demo Mode，讓 UI 可以在沒有 Python backend、robot、camera 的環境下執行 Create / Viewer / Pick workflow。第一版 mock data 可完成流程，但部分資料仍像合成 placeholder，demo 時不夠接近真實使用情境。

本功能將 Demo Mode 的主要資料來源改為 repo 內已 checked-in 的真實 fixture：

- Demo Mode saved model list 只使用 `demo_mode_data/saved_models/mycard` 與 `demo_mode_data/saved_models/pcb`
- 預設 saved model 使用 `demo_mode_data/saved_models/mycard`
- 第二個 saved model 使用 `demo_mode_data/saved_models/pcb`
- Create capture step 使用 `demo_mode_data/capure/image_1.png`、`image_2.png`、`image_3.png`
- Create build/review 3D feature data 使用 `demo_mode_data/capure/sift_3d_model.json`

目標是讓 UI designer 或 reviewer 在 Demo Mode 看到更真實的 capture image、saved model metadata、point cloud / feature points、pick pose 與 raw data，而不是純 synthetic placeholder。

此文件是 F06 的延伸需求。F06 仍定義 Demo Mode 的 mode switch、API facade、no-backend boundary 與 simulated async job 行為；本文件只定義 Demo Mode fixture data 的真實化要求。

---

## 2. 需求描述 (Requirement / User Story)

- **As a** UI designer / demo reviewer
- **I want** Demo Mode 使用 checked-in 的真實 capture/model fixture，而不是看起來像假資料的 placeholder
- **So that** 我可以更接近真實操作情境地檢查 Viewer、Create、Pick 的 UI flow、圖片呈現、點雲/feature 顯示、metadata 與 demo 效果

---

## 3. 功能範圍 (Scope)

### 3.1 In Scope

- Demo Mode Viewer Page 預設 saved model 使用 `mycard`。
- Demo Mode Viewer / Pick Page saved model list 只包含 `mycard` 與 `pcb`，不顯示 synthetic placeholder models。
- Demo Mode Pick Page 預設 pickable model list 包含 `mycard` 與 `pcb`。
- Demo Mode Create Page capture images 使用 `demo_mode_data/capure` 下的三張 PNG。
- Demo Mode Create Page build/review features 使用 `demo_mode_data/capure/sift_3d_model.json`。
- Demo Mode Viewer Page 對 `mycard` 顯示 realistic captures、point data、pick poses、raw JSON。
- Demo Mode 下 fixture assets 不得透過 `http://localhost:3001` 載入。
- Real Mode 行為保持不變。
- 保留 simulated async job behavior，讓 loading/progress/log UI 仍可被檢查。

### 3.2 Out Of Scope

- 修改 Python backend。
- 修改 backend `dry_run`。
- 將 `demo_mode_data/capure` 重新命名為 `capture`。
- 保證 fixture data 與真實 robot/camera kinematics 完全一致。
- 在 Demo Mode 進行永久資料寫入。
- 建立 production demo package。

---

## 4. Fixture Data Requirements

### 4.1 Saved Model Fixtures

Demo Mode 應只使用以下目錄作為 saved model fixtures：

```text
demo_mode_data/saved_models/mycard
demo_mode_data/saved_models/pcb
```

`mycard` 目前包含：

```text
demo_mode_data/saved_models/mycard/model.json
demo_mode_data/saved_models/mycard/sift_3d_model.json
demo_mode_data/saved_models/mycard/triangulated_points.json
demo_mode_data/saved_models/mycard/point_cloud_view.png
demo_mode_data/saved_models/mycard/captures/image_1.png
demo_mode_data/saved_models/mycard/captures/image_2.png
demo_mode_data/saved_models/mycard/captures/image_3.png
demo_mode_data/saved_models/mycard/captures/position_collected.json
demo_mode_data/saved_models/mycard/generated_masks/full_image_1_mask.png
demo_mode_data/saved_models/mycard/generated_masks/full_image_2_mask.png
demo_mode_data/saved_models/mycard/generated_masks/full_image_3_mask.png
demo_mode_data/saved_models/mycard/build_log.txt
demo_mode_data/saved_models/mycard/capture_log.txt
demo_mode_data/saved_models/mycard/pick_pose_log.txt
```

`model.json` 目前提供：

- `model_id = "mycard"`
- `model_name = "mycard"`
- `status = "built"`
- `latest_build.build_id`
- `latest_build.built_at`
- `latest_pick_pose.pick_name = "kao"`
- `latest_pick_pose.frame = "object_T_pick_tcp"`

`pcb/model.json` 目前提供：

- `model_id = "pcb"`
- `model_name = "PCB"`
- `status = "built"`
- `latest_build.build_id`
- `latest_build.built_at`
- `latest_pick_pose.pick_name = "default"`
- `latest_pick_pose.frame = "object_T_pick_tcp"`

Demo Mode 不應再顯示以下 synthetic placeholder saved models：

- `demo_tee_part`
- `demo_bracket_l`
- `demo_pcb_fixture`

### 4.2 Create Capture Fixture

Demo Mode Create Page capture step 應使用：

```text
demo_mode_data/capure/image_1.png
demo_mode_data/capure/image_2.png
demo_mode_data/capure/image_3.png
```

注意：repo 目前資料夾名稱是 `capure`，不是 `capture`。第一版應使用現有路徑，避免在此 feature 中混入資料夾 rename。

### 4.3 Create Build / Review Feature Fixture

Demo Mode Create Page build/review step 應使用：

```text
demo_mode_data/capure/sift_3d_model.json
```

此 JSON 可作為 `ModelCloudData`、build metadata、raw JSON tab 或 review summary 的資料來源。若 JSON shape 與 frontend 現有 TypeScript interface 不完全一致，mock layer 可做 normalization，但輸出應保持 deterministic。

---

## 5. 架構需求 (Architecture Requirements)

### 5.1 Demo API Boundary

Screens 仍應只 import `@/lib/api`。不要在 `CreateScreen.tsx`、`ViewerScreen.tsx`、`PickScreen.tsx` 中直接判斷 fixture path 或讀取 fixture JSON。

建議由以下檔案承接 fixture loading / normalization：

```text
app/src/renderer/src/lib/mockApi.ts
app/src/renderer/src/lib/mockData.ts
app/src/renderer/src/lib/demoMode.ts
```

### 5.2 Asset URL Requirement

Demo Mode assets 可以透過 frontend-safe URL、Vite import、public/static copy、data URL 或 Electron renderer 可讀路徑呈現，但不得變成 backend URL。

禁止：

```text
http://localhost:3001/...
```

允許：

```text
data:image/...
/assets/...
```

或其他不依賴 backend 的 renderer-safe asset URL。

### 5.3 Data Normalization

Fixture JSON 可轉換為現有 frontend API types：

- `ViewerModel`
- `PickableModel`
- `ModelCloudData`
- `CaptureInfo`
- `ModelDetail`
- `PickPoseSummary`
- `PickWorkflowResult`

Normalization 應保留 fixture 的核心資訊，例如 model id/name、built status、point counts、valid/unchecked/invalid counts、pick pose name、capture images、feature points。

---

## 6. 驗收準則 (Acceptance Criteria)

### Scenario 1: Viewer 預設顯示 mycard saved model

- **Given** Demo Mode 已啟用
- **When** 使用者進入 Viewer Page
- **Then** saved model list 只顯示 `mycard` 與 `pcb`
- **And** `mycard` 是初始選取的 model
- **And** model metadata 來自 `demo_mode_data/saved_models/mycard/model.json` 或由該 fixture normalization 而來

### Scenario 2: Viewer 顯示 mycard capture images

- **Given** Demo Mode 已啟用
- **And** Viewer Page 已選取 `mycard`
- **When** 使用者開啟 Source Images tab
- **Then** 顯示三張 capture images
- **And** image content 來自 `demo_mode_data/saved_models/mycard/captures/image_1.png` 到 `image_3.png`
- **And** image src 不包含 `http://localhost:3001`

### Scenario 3: Viewer 顯示 mycard point/feature data

- **Given** Demo Mode 已啟用
- **And** Viewer Page 已選取 `mycard`
- **When** 使用者開啟 Point Cloud tab 或 Raw JSON tab
- **Then** UI 顯示 deterministic point/feature data
- **And** point counts 與 valid/unchecked/invalid summary 來自 `mycard` fixture 或 normalization 結果

### Scenario 4: Pick Page 可使用 mycard

- **Given** Demo Mode 已啟用
- **When** 使用者進入 Pick Page
- **Then** pickable model list 只包含 `mycard` 與 `pcb`
- **And** 可選擇 `mycard` 的 pick pose
- **And** observe/detect 與 execute workflow 仍透過 simulated async job 完成

### Scenario 5: Create capture 使用真實 capture fixtures

- **Given** Demo Mode 已啟用
- **When** 使用者在 Create Page 執行 Start Capture
- **Then** capture job 仍以 `pollJob(job_id)` 顯示 progress/logs
- **And** job succeeded 後顯示 `demo_mode_data/capure/image_1.png`、`image_2.png`、`image_3.png`
- **And** image src 不包含 `http://localhost:3001`

### Scenario 6: Create build/review 使用 sift_3d_model fixture

- **Given** Demo Mode 已啟用
- **And** Create capture 已完成
- **When** 使用者執行 build/review flow
- **Then** build job 仍以 `pollJob(job_id)` 顯示 progress/logs
- **And** Review 或 point cloud display 使用 `demo_mode_data/capure/sift_3d_model.json` 的 deterministic feature data 或 normalized result

### Scenario 7: Real Mode 不受 fixture data 影響

- **Given** Demo Mode 未啟用
- **When** 使用者進入 Create / Viewer / Pick
- **Then** app 使用 real backend API helper
- **And** 不顯示 `demo_mode_data` fixture models
- **And** 現有 real API request body tests 保持通過

---

## 7. 測試情境 (Test Scenarios / Examples)

| ID | Scenario | Given | When | Then | Priority |
|----|----------|-------|------|------|----------|
| TC1 | Viewer saved model fixtures only | Demo Mode enabled | open Viewer Page | only `mycard` and `pcb` appear; `mycard` is selected by default | High |
| TC2 | Viewer mycard captures | Demo Mode enabled + `mycard` selected | open Source Images tab | 3 `mycard/captures/image_*.png` images render without localhost URL | High |
| TC3 | Viewer mycard point data | Demo Mode enabled + `mycard` selected | open Point Cloud tab | deterministic points render; counts match fixture-derived data | High |
| TC4 | Viewer raw fixture data | Demo Mode enabled + `mycard` selected | open Raw JSON tab | raw/normalized fixture metadata is visible | Medium |
| TC5 | Pick saved model fixtures only | Demo Mode enabled | open Pick Page | pickable model list is exactly `mycard` and `pcb` | High |
| TC6 | Pick mycard dry-run | Demo Mode enabled + `mycard` selected | run observe/detect | simulated job succeeds with realistic result data | High |
| TC7 | Create fixture captures | Demo Mode enabled | complete capture job | capture images come from `demo_mode_data/capure/image_*.png` | High |
| TC8 | Create fixture build data | Demo Mode enabled | complete build job | build/review data comes from `demo_mode_data/capure/sift_3d_model.json` or normalized output | High |
| TC9 | No backend asset URL leak | Demo Mode enabled | render Viewer/Create/Pick images | image src values do not contain `http://localhost:3001` | High |
| TC10 | Real API compatibility | Demo Mode disabled | run existing API tests | real backend request behavior remains unchanged | High |

---

## 8. 實作註記 (Implementation Notes)

### 8.1 Likely Affected Files

| File | Expected Change |
|------|-----------------|
| `app/src/renderer/src/lib/mockData.ts` | Load/normalize `mycard` and capture fixtures |
| `app/src/renderer/src/lib/mockApi.ts` | Return fixture-backed models/captures/cloud/results through existing API shape |
| `app/src/renderer/src/lib/demoMode.ts` | Ensure fixture asset URLs resolve without backend prefix |
| `app/src/renderer/src/lib/__tests__/mockApi.test.ts` | Add fixture-backed mock API tests |
| `app/src/renderer/src/lib/__tests__/apiFacadeDemo.test.ts` | Confirm demo facade does not call backend |
| `app/src/renderer/src/screens/__tests__/ViewerScreen.test.tsx` | Add or update demo fixture render assertions if screen-level coverage is practical |
| `app/src/renderer/src/screens/__tests__/CreateScreen.test.tsx` | Add or update capture fixture assertions if practical |
| `app/src/renderer/src/screens/__tests__/PickScreen.test.tsx` | Add or update `mycard` pickable workflow assertions if practical |

### 8.2 Fixture Access Strategy

The implementation should choose the simplest renderer-safe strategy supported by the app build:

- import/copy fixture images into a renderer-accessible asset path, or
- encode fixture PNGs as data URLs in mock data, or
- expose a frontend-only static asset path that does not use Python backend.

Whichever strategy is chosen, tests should verify no Demo Mode asset URL includes `http://localhost:3001`.

### 8.3 Preserve F06 Behavior

This feature must not break F06:

- `VITE_DEMO_MODE=1` remains the activation mechanism.
- `@/lib/api` remains the screen-facing facade.
- Mock async jobs still return `{ job_id }`.
- `pollJob(job_id)` still progresses through running states before `succeeded`.
- Real Mode remains unchanged.

### 8.4 Path Caveat

Use the repository's current path:

```text
demo_mode_data/capure
```

Do not silently correct it to `capture` in this feature. If the folder is renamed later, update this document, tests, and implementation paths together.

---

## 9. 補充說明 (Additional Notes)

### 9.1 Risks

- Large PNGs or JSON fixtures may need careful bundling to avoid slow test/build behavior.
- Browser/Electron renderer cannot read arbitrary repo files unless they are imported, copied, encoded, or served by the frontend toolchain.
- Fixture JSON may not exactly match current frontend interfaces; normalization should be tested.
- If fixture data is only partially used, the UI may still look synthetic in some tabs.

### 9.2 Related Documents

- `documents/implements/F06-Frontend-DemoMode.md`
- `documents/implements/frontend_demo_mode.md`

### 9.3 Documentation Follow-up

After implementation, update this document with:

- final fixture access strategy
- changed files
- test coverage
- commands run
- any fixture fields that were normalized or ignored

Also consider updating `documents/modules/GUI設計規劃.md` with the Demo Mode fixture data flow:

```text
Demo Mode:
Renderer -> Mock API -> demo_mode_data fixtures -> normalized frontend API types
```

---

## 附錄：TDD 需求實作流程提醒 (TDD Implementation Checklist)

請依照以下流程進行開發：

1. 先撰寫 fixture-backed mock API tests，確認 `mycard`、capture images、feature data、no-backend URL 行為。
2. 確認測試在實作前失敗。
3. 實作 fixture loading / normalization。
4. 讓 targeted tests 通過。
5. 跑 F06 相關 real/mock API tests 與 affected screen tests。
6. 跑 production build。
7. 更新本文件的 implementation record。

---

## Implementation Record

### Status

Implemented

### Implementation Summary

Demo Mode mock data now uses checked-in fixture data for the primary demo path:

- Viewer / Pick saved model list is exactly `mycard` and `pcb`.
- Viewer / Pick default saved model is `mycard`.
- `mycard` metadata is normalized from `demo_mode_data/saved_models/mycard/model.json`.
- `mycard` point cloud data is normalized from `demo_mode_data/saved_models/mycard/triangulated_points.json`.
- `mycard` captures use `demo_mode_data/saved_models/mycard/captures/image_1.png` through `image_3.png`.
- `pcb` metadata, captures, and point cloud data are normalized from `demo_mode_data/saved_models/pcb`.
- Synthetic saved models `demo_tee_part`, `demo_bracket_l`, and `demo_pcb_fixture` were removed from Demo Mode.
- Create capture jobs now populate captures from `demo_mode_data/capure/image_1.png` through `image_3.png`.
- Create build jobs now populate build counts and cloud points from `demo_mode_data/capure/sift_3d_model.json`.

Fixture PNGs are imported through Vite asset URLs (`?url`), so they are bundled into the renderer build as frontend assets and do not use `http://localhost:3001`. Fixture JSON is imported and normalized inside `mockData.ts`; screens still consume only the existing `@/lib/api` facade.

### Test Coverage

- `app/src/renderer/src/lib/__tests__/mockApi.test.ts`
  - Added F07 tests for `mycard` default model, `pcb` fixture model, exact saved model list, fixture captures, fixture-derived point counts, pickable availability, Create capture fixtures, and Create build fixture data.
- `app/src/renderer/src/lib/__tests__/apiFacadeDemo.test.ts`
  - Updated demo facade expectation so the demo model list is exactly `mycard` and `pcb`.
- Existing related suites remained passing:
  - `api.test.ts`
  - `ViewerScreen.test.tsx`
  - `CreateScreen.test.tsx`
  - `PickScreen.test.tsx`

### Files Changed

#### Production Files

- `app/src/renderer/src/lib/mockData.ts`
- `app/src/renderer/src/lib/mockApi.ts`
- `app/src/renderer/src/vite-env.d.ts`

#### Test Files

- `app/src/renderer/src/lib/__tests__/mockApi.test.ts`
- `app/src/renderer/src/lib/__tests__/apiFacadeDemo.test.ts`

#### Documentation Files

- `documents/implements/F07-Frontend-DemoMode-RealFixtureData.md`

### Acceptance Criteria Verification

| Acceptance Criterion | Status | Evidence |
| --- | --- | --- |
| Scenario 1: Viewer 預設顯示 mycard saved model | Passed | `mockApi.test.ts` F07 TC1 and follow-up fixture-only assertion |
| Scenario 2: Viewer 顯示 mycard capture images | Passed | `mockApi.test.ts` F07 TC2 |
| Scenario 3: Viewer 顯示 mycard point/feature data | Passed | `mockApi.test.ts` F07 TC3 |
| Scenario 4: Pick Page 可使用 mycard | Passed | `mockApi.test.ts` F07 TC5 and follow-up fixture-only pickable assertion |
| Scenario 5: Create capture 使用真實 capture fixtures | Passed | `mockApi.test.ts` F07 TC7-TC8 |
| Scenario 6: Create build/review 使用 sift_3d_model fixture | Passed | `mockApi.test.ts` F07 TC7-TC8 |
| Scenario 7: Real Mode 不受 fixture data 影響 | Passed | `api.test.ts` and full renderer test suite |

### FXX Test Case Verification

| FXX Test Case | Status | Automated Test Evidence |
| --- | --- | --- |
| TC1 Viewer saved model fixtures only | Passed | `mockApi.test.ts` F07 TC1 and follow-up fixture-only assertion |
| TC2 Viewer mycard captures | Passed | `mockApi.test.ts` F07 TC2 |
| TC3 Viewer mycard point data | Passed | `mockApi.test.ts` F07 TC3 |
| TC4 Viewer raw fixture data | Partial | Fixture metadata is normalized and available through `fetchModelDetail`; no separate Raw JSON screen assertion was added |
| TC5 Pick saved model fixtures only | Passed | `mockApi.test.ts` F07 TC5 and follow-up fixture-only assertion |
| TC6 Pick mycard dry-run | Passed | Existing mock pick workflow test now uses first pickable model, which is `mycard` |
| TC7 Create fixture captures | Passed | `mockApi.test.ts` F07 TC7-TC8 |
| TC8 Create fixture build data | Passed | `mockApi.test.ts` F07 TC7-TC8 |
| TC9 No backend asset URL leak | Passed | `mockApi.test.ts` F07 TC2 and TC7-TC8 |
| TC10 Real API compatibility | Passed | `api.test.ts` |

### Commands Run

```bash
npm test -- --run src/renderer/src/lib/__tests__/mockApi.test.ts
npm test -- --run src/renderer/src/lib/__tests__/apiFacadeDemo.test.ts src/renderer/src/lib/__tests__/api.test.ts src/renderer/src/screens/__tests__/ViewerScreen.test.tsx src/renderer/src/screens/__tests__/CreateScreen.test.tsx src/renderer/src/screens/__tests__/PickScreen.test.tsx
npm test -- --run src/renderer/src/lib/__tests__/apiFacadeDemo.test.ts src/renderer/src/lib/__tests__/api.test.ts src/renderer/src/lib/__tests__/mockApi.test.ts
npm test
npm run build
npx tsc -p tsconfig.web.json --noEmit
npm test -- --run src/renderer/src/lib/__tests__/mockApi.test.ts
npm test -- --run src/renderer/src/lib/__tests__/apiFacadeDemo.test.ts src/renderer/src/lib/__tests__/api.test.ts
npx tsc -p tsconfig.web.json --noEmit
npm run build
npm test
```

### Result

Red phase observed: F07 mock API tests initially failed because `mycard` was not present, `mycard` captures could not be fetched, point data did not come from fixtures, and created-model capture/build jobs still used synthetic data.

Green phase result:

- Targeted mock API tests: 12 passed.
- API-focused tests: 39 passed.
- Latest API/facade focused tests: 28 passed.
- Full renderer test suite: 7 files / 102 tests passed.
- Production build: passed.
- Web TypeScript check: passed.

### Assumptions

- Vite asset imports are the fixture access strategy for PNGs. Build output confirms the PNG assets are bundled into `out/renderer/assets`.
- JSON fixtures are imported directly and normalized in renderer code.
- `demo_mode_data/capure` remains intentionally misspelled to match the existing repository path.
- `triangulated_points.json` is used for `mycard` Viewer cloud data because it contains validation statuses and the expected built-model point summary.
- `triangulated_points.json` is also used for `pcb` Viewer cloud data.
- `sift_3d_model.json` is used for Create build/review fixture data because it contains the requested feature model counts and points.

### Deferred Items

- No separate screen-level Raw JSON tab assertion was added for TC4; fixture-derived detail/cloud behavior is covered at the mock API boundary and existing Viewer screen tests remain passing.
- No folder rename from `capure` to `capture`.
- No backend changes.

### Notes

The build bundles the imported PNG fixtures into renderer assets. Demo Mode image URLs stay frontend-owned and do not use the Python backend.
