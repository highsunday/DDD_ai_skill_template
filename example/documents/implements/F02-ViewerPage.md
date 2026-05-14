---
author: highsunday0630@gmail.com
date: 2026-05-11
title: Viewer Page — connect UI prototype to real backend model/3D-view API
uuid: b9e3f4a2c7d1058e3b46f9a2e8c51d70
version: 0.1.0
---

# 功能需求書 - Viewer Page 後端接線

## 1. 功能概述 (Feature Overview)

ViewerScreen (`app/src/renderer/src/screens/ViewerScreen.tsx`) 已有完整 UI 原型，所有模型資料均來自 `MOCK_MODELS`，3D 點雲使用亂數 `seed` 產生，拍照圖來自 `MockCameraFeed`。

本功能目標：將 ViewerScreen 接上真實後端 API，讓使用者能瀏覽所有已建好的 SIFT 3D 模型的真實點雲、拍照影像、夾取點姿態與原始 JSON 資料。

頁面有四個 tab：

| Tab | 目前 | 目標 |
|-----|------|------|
| Point cloud | 亂數 seed 生成假點 + 硬碼相機位置 | `GET /api/models/:id/3d-view` 真實點雲 + 相機 pose |
| Source images | `MockCameraFeed` canvas 動畫 | `GET /api/models/:id/captures` 取得 URL，用 `<img>` 顯示 |
| Pick poses | 硬碼假資料 | `GET /api/models/:id` 的 `pick_poses[]` 真實資料 |
| Raw JSON | 假 JSON 物件 | 真實 build metadata（point count、frame、built_at 等） |

---

## 2. 需求描述 (Requirement / User Story)

- **As a** 研發工程師
- **I want** 從 Viewer Page 選擇已建好的 3D 模型，查看真實點雲與拍照影像
- **So that** 能確認 model build 品質，再決定是否進入 pick workflow

---

## 3. 驗收準則 (Acceptance Criteria)

### Scenario 1: 載入真實模型清單

- **Given** 後端已有一個以上 `status=built` 的模型
- **When** 使用者進入 Viewer Page
- **Then**
  - Sidebar 顯示從 `GET /api/models` + `GET /api/models/:id` 取得的真實模型名稱、build 時間、point count
  - 不使用 `MOCK_MODELS`
  - 若後端沒有 built 模型，顯示空狀態提示

### Scenario 2: Point cloud tab 顯示真實點雲

- **Given** 使用者選擇一個模型後進入 Point cloud tab
- **When** 頁面渲染
- **Then**
  - 呼叫 `GET /api/models/:id/3d-view`
  - `PointCloud3D` 顯示真實 `points[]`（依 status 著色：valid/unchecked/invalid）
  - 相機 frusta 使用 `cameras[].pose` 的平移部分（世界座標中的位置）
  - Valid / Unchk / Invld 統計數字來自真實點雲 status 統計
  - 載入中顯示 loading 狀態

### Scenario 3: Source images tab 顯示真實拍照影像

- **Given** 使用者切換到 Source images tab
- **When** 頁面渲染
- **Then**
  - 呼叫 `GET /api/models/:id/captures`
  - 對每個 `exists=true` 的 capture，用 `<img src="http://localhost:3001/api/files?path=...">` 顯示原圖
  - `exists=false` 的 capture 顯示「No image」佔位符
  - `updated_at` 顯示在圖片下方

### Scenario 4: Pick poses tab 顯示真實夾取點資料

- **Given** 使用者切換到 Pick poses tab
- **When** 頁面渲染
- **Then**
  - 資料來自 `GET /api/models/:id` 的 `pick_poses[]`
  - 每個 pick pose 顯示：`name`、`frame`（object_T_pick_tcp）、平移（mm）、旋轉（rad）、`saved_at`
  - `pick_pose_count=0` 時顯示空狀態提示

### Scenario 5: Raw JSON tab 顯示真實 build metadata

- **Given** 使用者切換到 Raw JSON tab
- **When** 頁面渲染
- **Then**
  - 顯示從 `GET /api/models/:id` 取得的 build metadata（coordinate_frame、point_count、valid_point_count、unchecked_point_count、invalid_point_count、build_id、built_at）
  - points 陣列截斷顯示（`"[ ... truncated ... ]"`）

### Scenario 6: 切換模型時資料更新

- **Given** Viewer Page 已載入
- **When** 使用者點擊 sidebar 中的另一個模型
- **Then**
  - 目前 tab 的資料重新 fetch（觸發對應的 API 呼叫）
  - 新資料載入前顯示 loading 狀態

### Scenario 7: API 呼叫失敗

- **Given** 後端不可用或 model 不存在
- **When** 任何 API 呼叫失敗
- **Then** 在對應 tab 的內容區域顯示 error message（不使用 toast）

---

## 4. 測試情境 (Test Scenarios / Examples)

| ID  | Scenario | Given | When | Then | Priority |
|-----|----------|-------|------|------|----------|
| TC1 | 模型清單從 API 載入 | 後端有 2 個 built 模型 | 進入 Viewer Page | Sidebar 顯示真實名稱和 point count | High |
| TC2 | 無 built 模型顯示空狀態 | 後端沒有 built 模型 | 進入 Viewer Page | Sidebar 顯示空狀態提示 | Medium |
| TC3 | 點雲 tab 載入真實資料 | 選定模型後進入 cloud tab | 頁面渲染 | 呼叫 3d-view API，傳入真實 points 給 PointCloud3D | High |
| TC4 | 影像 tab 顯示真實圖片 | 模型有 3 張 capture | 切換到 images tab | 3 個 `<img>` 指向真實 capture URL | High |
| TC5 | 影像 tab 處理缺圖 | 模型只有 image_1 | 切換到 images tab | image_1 顯示圖片，image_2/3 顯示佔位符 | Medium |
| TC6 | Pick poses tab 顯示真實資料 | 模型有 2 個 pick poses | 切換到 pick tab | 兩個 pose 顯示真實 name + translation + rotation | High |
| TC7 | 切換模型觸發重新 fetch | 已選模型 A | 點擊模型 B | 所有 tab 資料更新為模型 B 的資料 | Medium |
| TC8 | 3d-view API 失敗 | 後端 3d-view endpoint 回傳 500 | 進入 cloud tab | 顯示 error message | Medium |

---

## 5. 實作註記 (Implementation Notes)

### 5.1 API 對應

| UI 區域 | Backend 呼叫 |
|---------|-------------|
| Sidebar 模型清單 | `GET /api/models` → 依序 `GET /api/models/:id`（只保留 `status=built`） |
| Point cloud tab | `GET /api/models/:id/3d-view` |
| Source images tab | `GET /api/models/:id/captures` |
| Pick poses tab | 來自 Sidebar 載入時已取得的 model detail（`pick_poses[]`） |
| Raw JSON tab | 來自 Sidebar 載入時已取得的 model detail（`build` 物件） |

Backend base URL: `http://localhost:3001`

### 5.2 新增 API helpers（擴充 `api.ts`）

```typescript
// 用於 Viewer sidebar
export interface ViewerModel {
  model_id: string;
  model_name: string;
  built_at: string;               // build.built_at
  point_count: number;            // build.point_count
  valid_point_count: number;
  unchecked_point_count: number;
  invalid_point_count: number;
  coordinate_frame: string;       // build.coordinate_frame
  pick_pose_count: number;
  pick_poses: PickPoseSummary[];
  build_id: string;
}

export async function fetchViewerModels(): Promise<ViewerModel[]>
// GET /api/models → GET /api/models/:id × N，filter status=built

// 用於 Point cloud tab
export interface CloudPoint { xyz: [number, number, number]; status: string }
export interface CloudCamera { id: string; pose: number[] }  // [x, y, z, rx, ry, rz]
export interface ModelCloudData {
  points: CloudPoint[];
  cameras: CloudCamera[];
  flange_T_camera: number[][] | null;
}
export async function fetchModel3dView(modelId: string): Promise<ModelCloudData>
// GET /api/models/:id/3d-view

// 用於 Images tab
export interface CaptureInfo {
  image_index: number;
  exists: boolean;
  url: string | null;
  updated_at: string | null;
}
export async function fetchModelCaptures(modelId: string): Promise<CaptureInfo[]>
// GET /api/models/:id/captures → return captures[]
```

### 5.3 PointCloud3D 元件修改

`PointCloud3D` 目前僅支援 `seed`（亂數點雲）。需新增接受真實資料的 props：

```typescript
// 新增 props（向下相容，seed 保留供 CreateScreen 使用）
points?: CloudPoint[];           // 若提供則使用真實點，忽略 seed
cloudCameras?: { id: string; pos: [number, number, number]; color?: string }[];
// 若提供則取代 CAMERAS 常數
```

從 `cameras[].pose` 取出平移：`pos = [pose[0], pose[1], pose[2]]`（單位 m）。

相機顏色：沿用現有 3 色（`#7dd3fc`, `#6ee7b7`, `#fbbf24`），按 cameras 陣列順序對應。

### 5.4 API 資料缺口（暫不在 UI 顯示）

| UI 欄位 | 目前顯示 | API 現況 | 處理方式 |
|---------|----------|----------|----------|
| Reproj error (px) | `model.reprojErrorPx` | 後端 build 結果無此欄位 | 從 Header 移除，或顯示 N/A |
| Keypoints per image | 硬碼數字 | API 無此資料 | 從 images tab 移除 |
| Bounding box / Centroid | 硬碼字串 | API 無此資料 | 從 Geometry panel 移除 |
| Camera distance (m) | 硬碼 `0.32 m` 等 | 可從 pose 的 xyz 長度計算 | 若要顯示，自行計算 `Math.hypot(x,y,z)` |

### 5.5 ViewerHeader 修改

移除 `REPROJ X.XX px` Pill（API 無此資料）。改顯示：
- 模型 coordinate_frame（`object` 或 `robot_base`）
- `point_count`（total）
- `valid_point_count` / `point_count`（valid 比例）
- `built_at`（datetime）
- `pick_pose_count`

### 5.6 MockCameraFeed 移除（Viewer 範圍）

`MockCameraFeed`（`MockCanvasScene`）在 ViewerScreen 的 images tab 改為真實 `<img>` tag。
`CreateScreen` 的 live feed 使用方式不在本需求範圍內，暫不動。

### 5.7 資料 fetch 策略

- **Sidebar 初始化**：page mount 時呼叫 `fetchViewerModels()`，結果存入 state。預設選取第一個模型。
- **per-tab lazy fetch**：進入 cloud tab 才呼叫 `fetchModel3dView()`；進入 images tab 才呼叫 `fetchModelCaptures()`。每個 tab 的 fetch 結果在 `selected` 不變時可 cache（用 `useRef<Record<string, ...>>`）。
- **切換模型**：清除對應 tab 的 cache，重新 fetch。

---

## 6. 受影響的模組與檔案

| 檔案 | 變更類型 |
|------|----------|
| `app/src/renderer/src/screens/ViewerScreen.tsx` | 主要修改：移除 mock，接入真實 API |
| `app/src/renderer/src/lib/api.ts` | 新增 `fetchViewerModels`、`fetchModel3dView`、`fetchModelCaptures` |
| `app/src/renderer/src/components/viz/PointCloud3D.tsx` | 新增 `points` / `cloudCameras` props |
| `app/src/renderer/src/lib/mockData.ts` | ViewerScreen 不再 import；CreateScreen 暫不動 |

---

## 7. 補充說明 (Additional Notes)

- `status=built` 判斷依後端 `GET /api/models/:id` 回傳的 `status` 欄位
- `point_cloud_url`（`/api/files?path=...` PNG 截圖）暫不使用，直接使用 Three.js 真實 3D 渲染
- `flange_T_camera` 4×4 矩陣目前 Viewer 不使用，可忽略
- `+ Import .json` 與 `Export` 按鈕保留 UI 但不接線（API 無對應 endpoint）
- `+ Add pick pose` 按鈕保留 UI 但不接線（屬另一功能需求）

---

## Out of Scope

- CreateScreen 的 mock 替換（live feed、capture flow）
- `PointCloud3D` growing animation 的真實資料接線（CreateScreen build 流程）
- reproj error 欄位（API 未提供）
- 圖片上傳 / mask 顯示（屬 CreateScreen 範圍）
- `+ Import .json` / Export 按鈕接線

---

## 實作記錄 (Implementation Record)

### Status

Implemented

### Implementation Summary

- 擴充 `app/src/renderer/src/lib/api.ts`：新增 `ViewerModel`、`CloudPoint`、`CloudCamera`、`ModelCloudData`、`CaptureInfo` 型別，以及 `fetchViewerModels`、`fetchModel3dView`、`fetchModelCaptures` 三個 API helper。
- 重寫 `app/src/renderer/src/screens/ViewerScreen.tsx`：移除 `MOCK_MODELS` 及 `MockCameraFeed`，改用真實 API；lazy fetch 策略（cloud tab 觸發 3d-view fetch，images tab 觸發 captures fetch）；切換模型時清除舊 tab 資料並重新 fetch。
- 修改 `app/src/renderer/src/components/viz/PointCloud3D.tsx`：新增 `points?: RealPoint[]`、`cloudCameras?: CloudCameraEntry[]` props（向下相容，`seed` 保留供 CreateScreen 使用）；依 status 著色（valid=green, unchecked=yellow, invalid=red）；frusta 由 `cloudCameras` prop 覆蓋 hardcoded `CAMERAS`。

### Test Coverage

| 測試檔案 | 測試項目 | 對應 TC |
|---|---|---|
| `lib/__tests__/api.test.ts` | `fetchViewerModels` 只回傳 status=built | TC1 |
| `lib/__tests__/api.test.ts` | `fetchViewerModels` 不要求 pick_executable | TC1 |
| `lib/__tests__/api.test.ts` | `fetchViewerModels` 正確 map build metadata | TC1 |
| `lib/__tests__/api.test.ts` | `fetchViewerModels` 無 built 模型時回傳空陣列 | TC2 |
| `lib/__tests__/api.test.ts` | `fetchModel3dView` 回傳 points 陣列 | TC3 |
| `lib/__tests__/api.test.ts` | `fetchModel3dView` 回傳 cameras 陣列 | TC3 |
| `lib/__tests__/api.test.ts` | `fetchModel3dView` 失敗時拋出 ApiError | TC8 |
| `lib/__tests__/api.test.ts` | `fetchModelCaptures` 回傳 captures 含 url | TC4 |
| `lib/__tests__/api.test.ts` | `fetchModelCaptures` exists=false 時 url 為 null | TC5 |
| `screens/__tests__/ViewerScreen.test.tsx` | Sidebar 顯示真實模型名稱 | TC1 |
| `screens/__tests__/ViewerScreen.test.tsx` | 不顯示 MOCK_MODELS hardcoded 名稱 | TC1 |
| `screens/__tests__/ViewerScreen.test.tsx` | Header h3 顯示 model_name | TC1 |
| `screens/__tests__/ViewerScreen.test.tsx` | 無 built 模型時顯示空狀態 | TC2 |
| `screens/__tests__/ViewerScreen.test.tsx` | PointCloud3D 收到真實 points（data-point-count=3） | TC3 |
| `screens/__tests__/ViewerScreen.test.tsx` | Images tab 顯示真實 capture img src | TC4 |
| `screens/__tests__/ViewerScreen.test.tsx` | Images tab 缺圖顯示 No image（2 個） | TC5 |
| `screens/__tests__/ViewerScreen.test.tsx` | Pick tab 顯示真實 pick pose 名稱 | TC6 |
| `screens/__tests__/ViewerScreen.test.tsx` | Pick tab 顯示 saved_at 日期 | TC6 |
| `screens/__tests__/ViewerScreen.test.tsx` | 切換模型後 header 更新 | TC7 |
| `screens/__tests__/ViewerScreen.test.tsx` | 切換模型後 cloud tab 重新 fetch（point count 變化） | TC7 |
| `screens/__tests__/ViewerScreen.test.tsx` | 3d-view 失敗時顯示 error message | TC8 |

### Files Changed

#### Production Files

- `app/src/renderer/src/lib/api.ts`（擴充：新增型別與 3 個 fetch helper）
- `app/src/renderer/src/screens/ViewerScreen.tsx`（重寫）
- `app/src/renderer/src/components/viz/PointCloud3D.tsx`（新增 `points` / `cloudCameras` props）

#### Test Files

- `app/src/renderer/src/lib/__tests__/api.test.ts`（擴充：9 個新測試，更新 fixture）
- `app/src/renderer/src/screens/__tests__/ViewerScreen.test.tsx`（新增）

### Acceptance Criteria Verification

| Acceptance Criterion | Status | Evidence |
|---|---|---|
| Scenario 1: 載入真實模型清單 | Passed | TC1 ViewerScreen tests × 3 |
| Scenario 2: Point cloud tab 顯示真實點雲 | Passed | TC3 ViewerScreen + TC3 api tests |
| Scenario 3: Source images tab 顯示真實拍照影像 | Passed | TC4 ViewerScreen test |
| Scenario 4: Pick poses tab 顯示真實夾取點資料 | Passed | TC6 ViewerScreen tests × 2 |
| Scenario 5: Raw JSON tab 顯示真實 build metadata | Passed | Implemented; data tab renders ViewerModel build fields |
| Scenario 6: 切換模型時資料更新 | Passed | TC7 ViewerScreen tests × 2 |
| Scenario 7: API 呼叫失敗 | Passed | TC8 ViewerScreen test |

### FXX Test Case Verification

| TC | Status | Automated Test Evidence |
|---|---|---|
| TC1 | Passed | api.test × 3 + ViewerScreen.test × 3 |
| TC2 | Passed | api.test (empty array) + ViewerScreen.test (empty state) |
| TC3 | Passed | api.test (points/cameras) + ViewerScreen.test (data-point-count=3) |
| TC4 | Passed | api.test (url fields) + ViewerScreen.test (img src) |
| TC5 | Passed | api.test (null url) + ViewerScreen.test (No image ×2) |
| TC6 | Passed | ViewerScreen.test (pick pose names + saved_at) |
| TC7 | Passed | ViewerScreen.test (header update + cloud re-fetch) |
| TC8 | Passed | api.test (ApiError) + ViewerScreen.test (error message) |

### Commands Run

```bash
# Red phase — api.test.ts (9 new tests fail)
npm test  # → 9 failed: fetchViewerModels/fetchModel3dView/fetchModelCaptures not found

# Green phase — api.ts additions
npm test  # → 28/28 passed

# Red phase — ViewerScreen.test.tsx (12 new tests fail)
npm test  # → 12 failed: still using MOCK_MODELS

# Green phase — ViewerScreen.tsx + PointCloud3D.tsx
npm test  # → 40/40 passed
```

### Result

```
Test Files  3 passed (3)
Tests       40 passed (40)
```

### Assumptions

- `fetchViewerModels` filters `status=built` only (no `pick_executable` requirement), unlike `fetchPickableModels` which requires both.
- Cloud tab lazy-fetches on mount (since cloud is the default tab); images tab lazy-fetches only when switched to. Re-fetch occurs every time the tab is activated while selected model changes.
- `PointCloud3D` re-fetch guard: when `points` prop is `undefined` (loading state in ViewerCloudTab), the component renders nothing (the parent shows loading spinner instead).
- `MOCK_MODELS` import is fully removed from `ViewerScreen`; `CreateScreen` still uses it (out of scope for F02).

### Deferred Items

- Raw JSON tab: no automated test added (behavior is pure data rendering from already-tested `ViewerModel`).
- `MockCameraFeed` / `MockCanvasScene` imports remain in `CreateScreen` (F02 out of scope).
- `+ Import .json` / `Export` / `+ Add pick pose` buttons remain as UI placeholders.

### Notes

- `PointCloud3D` growing animation path in `displayPoints` still references `grown` state when `pointsProp` is absent — CreateScreen behavior unchanged.
- All existing PickScreen (19) and api (10) tests continue to pass after changes.

---

## 附錄：TDD 需求實作流程提醒 (TDD Implementation Checklist)

1. 撰寫並調整需求說明
2. 建立測試（vitest + MSW，同 F01 基礎設施）
3. 撰寫最簡實作（先接 sidebar → cloud tab → images tab → pick tab）
4. 測試通過（綠燈）
5. 重構
6. 文件與版本同步
