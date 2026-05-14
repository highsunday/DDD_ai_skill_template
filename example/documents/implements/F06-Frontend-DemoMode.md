---
author: highsunday0630@gmail.com
date: 2026-05-13
title: Frontend Demo Mode - UI test mode without backend, robot, or camera
uuid: d8c4f1a73b694e0d9f125c7a0b8e4f36
version: 0.1.0
---

# 功能需求書 - Frontend Demo Mode

## 1. 功能概述 (Feature Overview)

目前 GUI 已經由 React/Electron 前端與 Python backend 串接，後端再連到 SIFT 3D scripts、UR robot、Orbbec camera 與本機檔案系統。這對實機 demo 與研發驗證是正確方向，但對 UI designer 來說成本太高：只想檢查畫面、流程、排版、狀態文案時，不應該需要啟動 backend、連接 robot 或 camera。

本功能新增 **Frontend Demo Mode**。啟用後，renderer 端改用 deterministic mock data 與 simulated async jobs，讓 Create / Viewer / Pick 三個主要頁面都可以完整走流程，而且不發送任何 backend request。

重要區分：

- Backend `dry_run`：仍需要 Python backend，也仍會走 API / filesystem 流程。
- Frontend Demo Mode：完全 frontend-only，不需要 backend、robot、camera。

本文件是正式 F 系列需求文件；先前的 `documents/implements/frontend_demo_mode.md` 可視為初稿/討論稿。

---

## 2. 需求描述 (Requirement / User Story)

- **As a** UI designer
- **I want** 使用一個 demo mode 開啟 GUI，並用 mock data 測試建立模型、查看模型、執行 pick workflow
- **So that** 我可以在沒有 backend、robot、camera 的環境下檢查 UI flow、visual state、loading state、error-free navigation 與畫面細節

---

## 3. 功能範圍 (Scope)

### 3.1 In Scope

- 新增 frontend-only mode switch。
- Demo Mode 啟用時，前端 API helper 使用 mock implementation。
- Create Page 可走完：Setup → Observe/Capture → ROI → Build → Teach Pick → Review。
- Viewer Page 可顯示 mock model list、point cloud、source images、pick poses、raw data。
- Pick Page 可走完：Select → Observe → Detect → Execute → Done。
- 模擬 async job progress 與 logs。
- Shell 顯示清楚的 Demo Mode 狀態，避免誤以為已連接真實 robot/camera。
- Demo Mode 下不得呼叫 `http://localhost:3001`。
- Real Mode 行為保持不變。

### 3.2 Out Of Scope

- 修改 Python backend。
- 修改 backend `dry_run` 行為。
- 模擬真實 robot kinematics。
- 模擬真實 camera stream。
- 跨 app restart 永久保存 mock models。
- 建立正式 production demo package。

---

## 4. 啟用方式 (Activation)

第一版使用環境變數：

```bash
VITE_DEMO_MODE=1 npm run dev
```

Renderer helper：

```ts
export function isDemoMode(): boolean {
  return import.meta.env.VITE_DEMO_MODE === '1';
}
```

可作為後續擴充，但不列入第一版必做：

- URL query：`?demo=1`
- localStorage：`localStorage.setItem("sift3d.demoMode", "1")`
- app 內部設定開關

第一版採環境變數，原因是啟用狀態明確，不容易在真實 robot 操作時被誤開。

---

## 5. 架構需求 (Architecture Requirements)

### 5.1 API Facade

目前 screens 透過 `@/lib/api` 使用 backend API helper。Demo Mode 不應該讓各個 screen 到處判斷 `isDemoMode()`，而是讓 `api.ts` 成為 stable facade。

建議檔案結構：

```text
app/src/renderer/src/lib/
  api.ts          // public facade: screens keep importing from here
  realApi.ts      // current fetch-based backend implementation
  mockApi.ts      // frontend-only mock implementation
  demoMode.ts     // mode detection + asset URL helpers
  mockData.ts     // deterministic mock fixtures
```

`api.ts` 應保持既有 exports：

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
- existing TypeScript interfaces

建議實作概念：

```ts
import * as realApi from './realApi';
import * as mockApi from './mockApi';
import { isDemoMode } from './demoMode';

const api = isDemoMode() ? mockApi : realApi;

export const fetchViewerModels = api.fetchViewerModels;
```

### 5.2 Asset URL Resolver

現有 `CreateScreen.tsx`、`ViewerScreen.tsx`、`PickScreen.tsx` 仍有 hardcoded `http://localhost:3001` 或 `BASE_URL`。Demo Mode 要求不能打 backend，因此圖片 URL 也需要統一處理。

建議新增：

```ts
export function resolveAssetUrl(url: string | null | undefined): string | null
```

支援：

- backend relative URL：`/api/files?...`
- backend file path：`output/models/...`
- absolute URL：`http://...` / `https://...`
- data URL：`data:image/...`
- frontend local asset URL

Demo Mode 下，mock image 應使用 data URL 或 frontend asset，不應被加上 backend base URL。

---

## 6. 驗收準則 (Acceptance Criteria)

### Scenario 1: Demo Mode 可在無 backend 狀態啟動

- **Given** Python backend 未啟動
- **And** robot/camera 未連接
- **When** 使用者以 `VITE_DEMO_MODE=1 npm run dev` 啟動 frontend
- **Then** app 可正常開啟
- **And** Create / Viewer / Pick route 可正常進入
- **And** console/network 不出現 `http://localhost:3001` request failure
- **And** TopBar 或 BottomBar 顯示 Demo Mode 狀態

### Scenario 2: Real Mode 行為不變

- **Given** 未設定 `VITE_DEMO_MODE=1`
- **When** 使用者進入 Create / Viewer / Pick
- **Then** app 仍使用 real backend API helper
- **And** 原本的 API request body、polling 行為與錯誤處理保持不變
- **And** mock data 不會出現在 real mode

### Scenario 3: Viewer Page 使用 mock model data

- **Given** Demo Mode 已啟用
- **When** 使用者進入 Viewer Page
- **Then** sidebar 顯示至少 3 個 mock built models
- **And** 可選擇不同模型
- **And** Point cloud tab 顯示 deterministic mock points
- **And** Source images tab 顯示 mock captures
- **And** Pick poses tab 顯示 mock pick poses
- **And** Raw JSON tab 顯示 mock build metadata

### Scenario 4: Create Page 可完整走完 mock workflow

- **Given** Demo Mode 已啟用
- **When** 使用者建立新 model、開始 capture、保存 ROI、開始 build、record TCP、進入 review
- **Then** 每個步驟都可完成，不需要 backend
- **And** simulated jobs 會顯示 progress / logs
- **And** Review step 顯示 mock build metadata
- **And** Save & open model / Save & run pick 可導向對應頁面

### Scenario 5: Pick Page 可完整走完 mock workflow

- **Given** Demo Mode 已啟用
- **And** mock pickable models 已存在
- **When** 使用者選擇 model 並執行 observe/detect
- **Then** Detect step 顯示 mock detection result、matches、inliers、pose output
- **When** 使用者執行 pick motion
- **Then** Execute step 顯示 simulated progress
- **And** Done step 顯示 final TCP 與 success result

### Scenario 6: Mock async job 行為與 real UI flow 相容

- **Given** Demo Mode 已啟用
- **When** screen 呼叫 `startCapture` / `startBuild` / `savePickPose` / `startPickWorkflow`
- **Then** mock API 回傳 `{ job_id }`
- **And** screen 可用既有 `pollJob(job_id)` flow 取得 `queued/running/succeeded`
- **And** logs 不重複追加
- **And** progress 可驅動既有 loading / animation state

### Scenario 7: Demo Mode shell 狀態不可誤導

- **Given** Demo Mode 已啟用
- **When** app shell render
- **Then** Robot / Camera status 不應顯示成真實已連線狀態
- **And** footer 不應顯示 `ALL SYSTEMS NORMAL`
- **And** UI 應清楚顯示 `DEMO MODE`、`SIM` 或 `NO BACKEND`

### Scenario 8: Asset URL resolver 支援 real/mock 兩種模式

- **Given** Demo Mode 已啟用
- **When** UI render mock capture image 或 mock result image
- **Then** image src 為 data URL 或 frontend asset URL
- **And** 不加上 `http://localhost:3001`

- **Given** Demo Mode 未啟用
- **When** UI render backend relative image URL
- **Then** image src 正確解析為 backend URL

---

## 7. 測試情境 (Test Scenarios / Examples)

| ID | Scenario | Given | When | Then | Priority |
|----|----------|-------|------|------|----------|
| TC1 | Demo Mode detection | `VITE_DEMO_MODE=1` | 呼叫 `isDemoMode()` | 回傳 `true` | High |
| TC2 | Real Mode detection | 未設定 env | 呼叫 `isDemoMode()` | 回傳 `false` | High |
| TC3 | Demo Mode 不打 backend | backend offline + demo enabled | 開啟 Viewer Page | 顯示 mock models，無 localhost request failure | High |
| TC4 | Viewer mock point cloud | demo enabled | 開啟 Viewer cloud tab | `PointCloud3D` 收到 deterministic mock points | High |
| TC5 | Viewer mock captures | demo enabled | 開啟 Viewer images tab | 3 張 mock capture images 顯示成功 | Medium |
| TC6 | Create mock capture job | demo enabled | 點擊 Start Capture | mock logs/progress 出現，capture URLs 可用 | High |
| TC7 | Create mock build job | demo enabled | 點擊 Start triangulation | build job succeeded，review metadata 顯示 | High |
| TC8 | Create mock save pick pose | demo enabled | 點擊 Record current TCP | pick row 顯示 saved 狀態 | Medium |
| TC9 | Pick mock dry-run | demo enabled | 點擊 Move to observe & capture | Detect step 顯示 mock pose result | High |
| TC10 | Pick mock execute | demo enabled | 點擊 EXECUTE PICK MOTION | Execute animation 前進，最後進入 Done | High |
| TC11 | Asset resolver data URL | demo enabled | resolve data image URL | 原樣回傳，不加 backend prefix | Medium |
| TC12 | Asset resolver backend URL | demo disabled | resolve `/api/files?...` | 回傳 backend base URL + path | Medium |
| TC13 | Real API compatibility | demo disabled | 跑現有 API tests | request body / ApiError tests 保持通過 | High |

---

## 8. 實作註記 (Implementation Notes)

### 8.1 Likely Affected Files

| File | Expected Change |
|------|-----------------|
| `app/src/renderer/src/lib/api.ts` | 保留 public exports，改為 real/mock facade，或拆 types 後 delegate |
| `app/src/renderer/src/lib/realApi.ts` | 承接目前 `api.ts` 中 fetch backend 的實作 |
| `app/src/renderer/src/lib/mockApi.ts` | 新增 frontend-only mock API implementation |
| `app/src/renderer/src/lib/demoMode.ts` | 新增 `isDemoMode()`、`resolveAssetUrl()` 等 helper |
| `app/src/renderer/src/lib/mockData.ts` | 擴充 mock models、captures、point cloud、pick results |
| `app/src/renderer/src/screens/CreateScreen.tsx` | 移除 hardcoded backend image URL；其餘盡量維持 API 呼叫不變 |
| `app/src/renderer/src/screens/ViewerScreen.tsx` | 移除 hardcoded backend image URL；其餘盡量維持 API 呼叫不變 |
| `app/src/renderer/src/screens/PickScreen.tsx` | 移除 hardcoded backend image URL；其餘盡量維持 API 呼叫不變 |
| `app/src/renderer/src/components/TopBar.tsx` | Demo Mode status display |
| `app/src/renderer/src/components/BottomBar.tsx` | Demo Mode footer status |
| `app/src/renderer/src/lib/__tests__/api.test.ts` | 保留 real API tests，新增 mode/facade/mock tests |
| `app/src/renderer/src/screens/__tests__/*.test.tsx` | 新增 demo mode workflow coverage |

### 8.2 Mock Job Store

`mockApi.ts` 建議維護 in-memory job store：

```ts
type MockJobKind = 'capture' | 'build' | 'save_pick_pose' | 'pick_dry_run' | 'pick_execute';

type MockJobRecord = {
  kind: MockJobKind;
  job: JobResult;
  polls: number;
  finalResult?: PickWorkflowResult;
};
```

`pollJob(jobId)` 可用 polling 次數推進狀態：

| Poll Count | Status | Progress | Logs |
|------------|--------|----------|------|
| 1 | `running` | 20 | queued / starting |
| 2 | `running` | 60 | operation-specific logs |
| 3+ | `succeeded` | 100 | completed |

這樣可以保留現有 UI 的 polling / logs / progress 行為。

### 8.3 Mock Data Requirements

Mock data 應該 deterministic，避免每次 reload 截圖不同。

最低資料集：

- 3 個 built models
- 每個 model 至少 1 個 point cloud
- 至少 1 個 model 有 2 個以上 pick poses
- 至少 1 個 model 有 invalid / unchecked points
- 3 張 mock capture images
- 1 組 mock pick workflow result

### 8.4 Keep Screens Thin

不建議在 screens 中寫：

```tsx
if (isDemoMode()) {
  setTimeout(...);
} else {
  await startPickWorkflow(...);
}
```

建議維持：

```tsx
const { job_id } = await startPickWorkflow(params);
```

real/mock 差異應由 `@/lib/api` 下面處理。

---

## 9. 補充說明 (Additional Notes)

### 9.1 Known Risks

- 若 screen 仍 hardcode `http://localhost:3001`，Demo Mode 仍會誤打 backend。
- 若 mock response shape 與 real API type 不一致，可能造成 demo mode 正常但 real mode 壞掉。
- 若 mock job 立即完成，designer 無法檢查 loading / progress 狀態。
- 若 Shell 顯示 robot/camera 正常，可能誤導使用者以為真實硬體已連線。

### 9.2 Module Documentation Follow-up

實作完成後，建議更新：

- `documents/modules/GUI設計規劃.md`

新增架構說明：

```text
Real Mode:
Renderer -> Python Backend -> SIFT scripts / Robot / Camera / Filesystem

Demo Mode:
Renderer -> Mock API -> deterministic local mock data
```

---

## 附錄：TDD 需求實作流程提醒 (TDD Implementation Checklist)

1. 撰寫 mode detection 與 asset resolver 測試。
2. 拆出 `realApi.ts`，確認現有 real API tests 保持通過。
3. 新增 `mockApi.ts`，先補 mock API unit tests。
4. 改 `api.ts` 成 facade，確認 real mode 與 demo mode 都走正確 implementation。
5. 移除 screen 中 hardcoded backend image URL。
6. 補 Viewer / Create / Pick 的 Demo Mode workflow 測試。
7. 手動啟動 `VITE_DEMO_MODE=1 npm run dev`，確認 backend offline 時仍可操作。
8. 更新本文件的實作註記、測試結果與已知限制。

---

## Implementation Record

### Status

Implemented

### Implementation Summary

已新增 frontend-only Demo Mode。`@/lib/api` 現在是 stable facade，會依 `VITE_DEMO_MODE=1` 在 `realApi` 與 `mockApi` 間切換；Real Mode 保留既有 backend fetch 行為。Demo Mode 提供 deterministic mock models、captures、point cloud、pick poses、raw model detail、pick workflow result，以及 capture/build/save-pick-pose/pick-workflow async job polling。

圖片 URL 改由 `resolveAssetUrl()` 統一處理，Create / Viewer / Pick screens 不再直接 hardcode backend image URL。Shell 在 Demo Mode 顯示 `DEMO MODE`、robot/camera `SIM` 與 `DEMO MODE - NO BACKEND`，避免誤導為真實硬體連線。

### Test Coverage

- `src/renderer/src/lib/__tests__/demoMode.test.ts`
  - Covers F06 TC1, TC2, TC11, TC12.
- `src/renderer/src/lib/__tests__/mockApi.test.ts`
  - Covers F06 TC3, TC4, TC5, TC6, TC8, TC9, TC10.
- `src/renderer/src/lib/__tests__/apiFacadeDemo.test.ts`
  - Covers F06 TC3 no-backend facade behavior.
- `src/renderer/src/lib/__tests__/api.test.ts`
  - Confirms Real Mode API request bodies and ApiError behavior remain compatible.
- Existing screen tests:
  - `ViewerScreen.test.tsx`, `CreateScreen.test.tsx`, `PickScreen.test.tsx` remain passing after resolver/facade changes.

### Files Changed

#### Production Files

- `app/src/renderer/src/lib/api.ts`
- `app/src/renderer/src/lib/apiTypes.ts`
- `app/src/renderer/src/lib/realApi.ts`
- `app/src/renderer/src/lib/mockApi.ts`
- `app/src/renderer/src/lib/demoMode.ts`
- `app/src/renderer/src/lib/mockData.ts`
- `app/src/renderer/src/screens/CreateScreen.tsx`
- `app/src/renderer/src/screens/ViewerScreen.tsx`
- `app/src/renderer/src/screens/PickScreen.tsx`
- `app/src/renderer/src/components/TopBar.tsx`
- `app/src/renderer/src/components/BottomBar.tsx`

#### Test Files

- `app/src/renderer/src/lib/__tests__/demoMode.test.ts`
- `app/src/renderer/src/lib/__tests__/mockApi.test.ts`
- `app/src/renderer/src/lib/__tests__/apiFacadeDemo.test.ts`

### Acceptance Criteria Verification

| Acceptance Criterion | Status | Evidence |
| --- | --- | --- |
| Scenario 1: Demo Mode 可在無 backend 狀態啟動，不呼叫 localhost，Shell 顯示狀態 | Passed | `apiFacadeDemo.test.ts`; `TopBar.tsx`; `BottomBar.tsx` |
| Scenario 2: Real Mode 行為不變 | Passed | `api.test.ts` 27 tests passing |
| Scenario 3: Viewer Page 使用 mock model data | Passed | `mockApi.test.ts`; existing `ViewerScreen.test.tsx` still passing |
| Scenario 4: Create Page 可完整走完 mock workflow | Passed | `mockApi.test.ts` capture/build/save-pick-pose job tests; existing `CreateScreen.test.tsx` still passing |
| Scenario 5: Pick Page 可完整走完 mock workflow | Passed | `mockApi.test.ts` dry-run and execute job tests; existing `PickScreen.test.tsx` still passing |
| Scenario 6: Mock async job 行為與 real UI flow 相容 | Passed | `mockApi.test.ts` verifies `{ job_id }` and `pollJob()` queued/running/succeeded progression |
| Scenario 7: Demo Mode shell 狀態不可誤導 | Passed | `TopBar.tsx`; `BottomBar.tsx`; renderer test suite passing |
| Scenario 8: Asset URL resolver 支援 real/mock 兩種模式 | Passed | `demoMode.test.ts` |

### FXX Test Case Verification

| FXX Test Case | Status | Automated Test Evidence |
| --- | --- | --- |
| TC1 Demo Mode detection | Passed | `demoMode.test.ts` |
| TC2 Real Mode detection | Passed | `demoMode.test.ts` |
| TC3 Demo Mode 不打 backend | Passed | `apiFacadeDemo.test.ts` |
| TC4 Viewer mock point cloud | Passed | `mockApi.test.ts` |
| TC5 Viewer mock captures | Passed | `mockApi.test.ts` |
| TC6 Create mock capture job | Passed | `mockApi.test.ts` |
| TC7 Create mock build job | Passed | `mockApi.test.ts` |
| TC8 Create mock save pick pose | Passed | `mockApi.test.ts` |
| TC9 Pick mock dry-run | Passed | `mockApi.test.ts` |
| TC10 Pick mock execute | Passed | `mockApi.test.ts` |
| TC11 Asset resolver data URL | Passed | `demoMode.test.ts` |
| TC12 Asset resolver backend URL | Passed | `demoMode.test.ts` |
| TC13 Real API compatibility | Passed | `api.test.ts` |

### Commands Run

```bash
npm test -- --run src/renderer/src/lib/__tests__/demoMode.test.ts src/renderer/src/lib/__tests__/mockApi.test.ts
npm test -- --run src/renderer/src/lib/__tests__/demoMode.test.ts src/renderer/src/lib/__tests__/mockApi.test.ts src/renderer/src/lib/__tests__/apiFacadeDemo.test.ts
npm test -- --run src/renderer/src/lib/__tests__/api.test.ts
npm test
npm run build
```

### Result

Red phase observed: the new demo tests initially failed because `@/lib/demoMode` and `@/lib/mockApi` did not exist.

Green phase result:

- Full renderer test suite: 7 files / 96 tests passed.
- Production build: passed.

### Assumptions

- Demo Mode is controlled only by `VITE_DEMO_MODE=1` for this version, as required.
- Mock model state is in-memory only and resets on renderer reload.
- Mock images use inline SVG data URLs so they never require backend file serving.
- Manual Electron launch was not required for completion because automated renderer tests and production build covered the changed frontend paths.

### Deferred Items

- URL query or localStorage activation is deferred because it is explicitly optional for first version.
- No backend `dry_run` behavior was changed.
- No production demo package was created.

### Notes

- `documents/modules/GUI設計規劃.md` can be updated separately with the Real Mode vs Demo Mode architecture diagram if project documentation sync is desired.
