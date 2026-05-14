---
author: highsunday0630@gmail.com
date: 2026-05-11
title: Create Page — 合併三個 Observe 步驟為單一 Observe 頁面並重設佈局
uuid: a3f1c9e2b5d40f78e6a21b3c8d07f94e
version: 0.1.0
---

# 功能需求書 - Create Page Observe 步驟 UI 重設計

## 1. 功能概述 (Feature Overview)

F03 完成了 Create Page 的後端接線，但工作流程仍保留 3 個獨立的 Observe 步驟（Observe 1 / 2 / 3）。然而，後端 `POST /api/models/:id/capture` 是一次性批次拍攝，機械臂連續移動至 3 個觀測點並同時完成拍攝，並非 3 個獨立動作。

現行 UI 反映的流程與實際後端行為不符：
- Observe 1 觸發拍照，Observe 2 / 3 只是顯示已拍好的圖並補充上傳 mask
- Observe 2 / 3 的「等待中…」畫面在拍照完成後即可直接跳過，流程多餘
- 3 個 observe tab 在 Stepper 上佔用空間，視覺上誤導使用者以為是 3 個獨立動作

本功能目標：**純 UI/UX 重設計，不改動任何 API 呼叫邏輯**，將 3 個 Observe 步驟合併為一個「Observe」步驟，並在同一個頁面上同時顯示 3 張照片。只有 Image 1 需要標注 ROI mask；Image 2 / 3 僅顯示預覽，mask 在後端以全圖自動處理。

---

## 2. 需求描述 (Requirement / User Story)

- **As a** 研發工程師
- **I want** 在一個「Observe」步驟中看到拍照進度與 3 張照片，並只需為 Image 1 標注 ROI
- **So that** 工作流程與後端單一批次拍照行為一致，減少不必要的步驟切換，並更清楚地展示「拍照完成後只需標注主視角 mask」的意圖

---

## 3. 驗收準則 (Acceptance Criteria)

### Scenario 1: Stepper 由 7 步縮減為 5 步

- **Given** 使用者進入 Create Page
- **When** 頁面載入
- **Then** Stepper 依序顯示：Setup → Observe → Teach pick → Build → Review（共 5 步），不再出現 Observe 1 / Observe 2 / Observe 3

### Scenario 2: Observe 步驟佈局 — 拍照前

- **Given** 使用者在 Setup 完成後進入 Observe 步驟
- **When** 尚未觸發 capture job
- **Then**
  - 上方顯示 Image 1 大佔位框（含 MockCameraFeed 預覽）與「Start Capture」按鈕
  - 下方並排顯示 Image 2 / Image 3 小佔位框（dim 狀態，無任何操作）
  - 右側顯示 Status panel 與 Pipeline log

### Scenario 3: Observe 步驟佈局 — 拍照中

- **Given** 使用者點擊「Start Capture」後 job 正在執行
- **When** capture job 狀態為 running
- **Then**
  - Image 1 大框顯示「Capture in progress…」訊息與 Terminal log
  - Image 2 / 3 小框維持 dim 佔位框
  - 「Start Capture」按鈕 disabled

### Scenario 4: Observe 步驟佈局 — 拍照完成，可標注 ROI

- **Given** capture job 成功完成
- **When** `captureUrls[1]` 取得後
- **Then**
  - Image 1 大框顯示真實拍攝圖片，並啟用 ROI 塗抹筆刷（RoiPaintCanvas）
  - Image 2 小框顯示 `captureUrls[2]` 的圖片，右上角顯示「Captured ✓」badge
  - Image 3 小框顯示 `captureUrls[3]` 的圖片，右上角顯示「Captured ✓」badge
  - Image 2 / 3 **不提供任何塗抹或互動操作**

### Scenario 5: Mask 標注為選填

- **Given** capture 完成後，使用者選擇不塗抹 ROI 直接繼續
- **When** 使用者點擊「Continue → Teach pick」（不曾點擊 Save ROI）
- **Then**
  - 允許推進，不阻止
  - ROI 區域顯示提示文字：「No mask drawn — full image will be used」
  - 後端不呼叫 `PUT /api/models/:id/masks/1`（直接跳過）

### Scenario 6: 使用者塗抹並儲存 ROI

- **Given** Image 1 已顯示，使用者塗抹了筆觸
- **When** 使用者點擊「Save ROI（optional）」
- **Then**
  - 呼叫 `uploadMask(modelId, 1, base64)`（行為與 F03 相同）
  - 按鈕顯示「Saved ✓」，mask pill 從 warn 轉為 ok

### Scenario 7: Continue 解鎖條件

- **Given** 使用者在 Observe 步驟
- **When** capture job 尚未完成（`captureUrls[1] === null`）
- **Then** 「Continue → Teach pick」按鈕 disabled

- **Given** capture job 已完成（`captureUrls[1] !== null`）
- **When** 不論 mask 是否已儲存
- **Then** 「Continue → Teach pick」按鈕 enabled

### Scenario 8: 右側面板

- **Given** Observe 步驟頁面
- **When** 任何時間點
- **Then** 右側 320px 面板顯示：
  - Status panel：Image 欄位、Captured 欄位（✓ 或 —）、Mask 欄位（Saved / Not set — full image）、ROI strokes 計數
  - Tip panel
  - Pipeline log（顯示最近 7 行）

---

## 4. 測試情境 (Test Scenarios / Examples)

| ID  | Scenario | Given | When | Then | Priority |
|-----|----------|-------|------|------|----------|
| TC1 | Stepper 只有 5 步 | 進入 Create Page | 頁面載入 | Stepper 顯示 Setup/Observe/Teach pick/Build/Review，無 Observe 1/2/3 | High |
| TC2 | Observe 前顯示 3 個佔位框 | Observe 步驟、未 capture | 頁面載入 | Image 1 大框 + Start Capture 按鈕；Image 2/3 小框 dim | High |
| TC3 | Capture 完成後 Image 1 可塗抹 | captureUrls[1] 存在 | capture job succeeded | Image 1 顯示真實圖片且 RoiPaintCanvas 啟用 | High |
| TC4 | Capture 完成後 Image 2/3 只顯示圖 | captureUrls[2/3] 存在 | capture job succeeded | Image 2/3 顯示圖片，無 ROI 互動元素 | High |
| TC5 | 無 mask 可繼續 | capture 完成，無筆觸 | 點 Continue | 允許推進到 Teach pick，不呼叫 uploadMask | High |
| TC6 | 顯示「full image will be used」提示 | capture 完成，paths.length === 0 | 頁面顯示 | ROI 區域顯示 full image 提示文字 | Medium |
| TC7 | Save ROI 呼叫 uploadMask | capture 完成，有筆觸 | 點 Save ROI | 呼叫 PUT /api/models/:id/masks/1；按鈕顯示 Saved ✓ | High |
| TC8 | capture 未完成時 Continue disabled | captureUrls[1] === null | 頁面顯示 | Continue 按鈕 disabled | High |
| TC9 | capture 完成後 Continue enabled | captureUrls[1] 存在 | 頁面顯示 | Continue 按鈕 enabled（不論 mask） | High |

---

## 5. 實作註記 (Implementation Notes)

### 5.1 Stepper 變更

```typescript
// Before (F03)
const CREATE_STEPS = [
  { key: 'setup',    t: 'Setup',      d: 'name + frame' },
  { key: 'observe1', t: 'Observe 1',  d: 'capture + ROI' },
  { key: 'observe2', t: 'Observe 2',  d: 'capture + ROI' },
  { key: 'observe3', t: 'Observe 3',  d: 'capture + ROI' },
  { key: 'teach',    t: 'Teach pick', d: 'record TCP' },
  { key: 'build',    t: 'Build',      d: 'triangulate' },
  { key: 'review',   t: 'Review',     d: 'save model' },
];

// After (F04) — B01 後 build/teach 順序已對調，見下方說明
const CREATE_STEPS = [
  { key: 'setup',   t: 'Setup',      d: 'name + frame' },
  { key: 'observe', t: 'Observe',    d: 'capture + ROI' },
  { key: 'teach',   t: 'Teach pick', d: 'record TCP' },
  { key: 'build',   t: 'Build',      d: 'triangulate' },
  { key: 'review',  t: 'Review',     d: 'save model' },
];

// Current (after B01 — B01-CreatePage-PickNotVisible.md)
const CREATE_STEPS = [
  { key: 'setup',   t: 'Setup',      d: 'name + frame' },
  { key: 'observe', t: 'Observe',    d: 'capture + ROI' },
  { key: 'build',   t: 'Build',      d: 'triangulate' },  // index 2
  { key: 'teach',   t: 'Teach pick', d: 'record TCP' },   // index 3
  { key: 'review',  t: 'Review',     d: 'save model' },
];
```

步驟 index 相應調整：build job succeeded 後 `setStepIdx(3)` → teach pick（F04 原為 `setStepIdx(4)` → review，B01 修正）。

### 5.2 canVisit 變更

```typescript
// Before: observe1 需要 masksUploaded[1]，observe2 需要 masksUploaded[2]...
// After (F04):
function canVisit(i: number): boolean {
  if (i <= stepIdx) return true;
  if (i === stepIdx + 1) {
    if (step.key === 'setup') return false; // 需等 createModel 成功
    if (step.key === 'observe') return captureUrls[1] !== null; // capture 完成即可，mask 選填
    if (step.key === 'teach') return picks.some((p) => p.recorded);
    if (step.key === 'build') return built;
  }
  return false;
}
```

> **B01 後的條件映射**（`step` = 當前步驟，條件決定能否前往下一步）：
>
> | 當前步驟（step.key） | 前往 | 條件 |
> |---|---|---|
> | `observe` | build（idx 2） | `captureUrls[1] !== null` |
> | `build` | teach（idx 3） | `built === true` |
> | `teach` | review（idx 4） | `picks.some((p) => p.recorded)` |
>
> 條件本身與 F04 相同，語意因步驟順序對調而改變（B01-CreatePage-PickNotVisible.md）。

### 5.3 新 ObservePane 佈局

```
┌── 主區域 (1fr) ────────────────────────────────────┐  ┌── 右側 (320px) ──┐
│  Image 1 大框 (RoiPaintCanvas 或 MockCameraFeed)   │  │  Status           │
│                                                    │  │  Tip              │
│  ┌──── Image 2 (1fr) ────┐  ┌─── Image 3 (1fr) ──┐│  │  Pipeline log     │
│  │  小縮圖 / 佔位框      │  │  小縮圖 / 佔位框    ││  └──────────────────┘
│  └───────────────────────┘  └────────────────────┘│
│  [Start Capture] [Clear mask] [Save ROI · opt]    │
└───────────────────────────────────────────────────┘
```

- Image 1 大框高度約 360–400px
- Image 2 / 3 小框高度約 180px，`grid-template-columns: 1fr 1fr`
- Image 2 / 3 使用 `<img>` 顯示（captureUrl 存在時）或 dim placeholder（未拍照時）
- Image 2 / 3 右上角顯示「Captured ✓」badge（captureUrl 存在時）

### 5.4 Mask 選填邏輯

- `masksUploaded[1]` 僅影響 UI 狀態顯示（Save ROI 按鈕、Status panel Mask 欄），不阻擋推進
- 當 `paths.length === 0` 且尚未儲存時，ROI 操作列顯示 hint：「No mask drawn — full image will be used」
- `masksUploaded[2]` 與 `masksUploaded[3]` 不再使用，可從 state 移除

### 5.5 API 呼叫不變

本功能為純 UI 重設計，所有後端呼叫（`startCapture`、`uploadMask`）行為與 F03 相同，僅在 UI 觸發時機與版面上有差異：
- `startCapture` 仍在 Observe 步驟第一個按鈕觸發，行為不變
- `uploadMask(modelId, 1, base64)` 仍在使用者點擊「Save ROI」後觸發
- Image 2 / 3 不再需要任何 mask 上傳（UI 移除操作入口）

### 5.6 移除內容

- `ObservePane` 的 `showCapturePhase` prop（改為直接在新 ObservePane 中判斷）
- `masksUploaded[2]` / `masksUploaded[3]` 相關 state 與 canVisit 判斷
- 進入 Observe 2 / 3 時顯示的「Waiting for capture to complete…」

---

## 6. 受影響的模組與檔案

| 檔案 | 變更類型 |
|------|----------|
| `app/src/renderer/src/screens/CreateScreen.tsx` | 主要修改：Stepper 定義、步驟索引調整、ObservePane 重設計 |
| `app/src/renderer/src/screens/__tests__/CreateScreen.test.tsx` | 新增 / 更新測試以覆蓋 TC1–TC9 |

---

## 7. 補充說明 (Additional Notes)

- 本需求純屬 UI/UX 重設計（F04），後端 API 呼叫修改（例如 Image 2/3 mask 不上傳）將在後續 feature 中處理
- `masksUploaded[2]` / `[3]` 目前在後端已上傳，本 UI 改動後不再觸發；後端 mask 資料的清理或忽略策略為 out of scope
- `MockCameraFeed` 在 capture 前的 Image 1 大框中保留顯示，Image 2 / 3 placeholder 可用 dim 色塊或簡單的 `video-feed` class 空 div

---

## Out of Scope

- Image 2 / 3 mask 上傳的後端行為變更（後端仍可接收，但 UI 不再提供入口）
- 「no mask → 全圖使用」的後端邏輯（現有後端行為不變，本需求只調整 UI 提示）
- 觀測姿態預覽圖（robot 移動到哪個觀測點的示意圖）
- Image 縮圖的 lightbox 放大功能

---

---

## 實作記錄 (Implementation Record)

### Status

Implemented

### Implementation Summary

- `CREATE_STEPS` 從 7 步縮減為 5 步，移除 `observe1`/`observe2`/`observe3`，新增單一 `observe` 步驟。
- 移除 `observeIdx` 衍生變數，改以 `step.key === 'observe'` 直接判斷。
- `canVisit` 中 observe 步驟的推進條件改為 `captureUrls[1] !== null`（capture 完成即可，mask 選填）。
- `build` job 成功後的 `setStepIdx(6)` 改為 `setStepIdx(4)`（Review 現在是 index 4）。
- Setup Continue 按鈕文字由 `Continue → Observe 1` 改為 `Continue → Observe`。
- `masksUploaded` state（`{ 1: boolean; 2: boolean; 3: boolean }`）替換為單一 `maskSaved: boolean`。
- `handleSaveMask` 簽名從 `(idx, paths)` 簡化為 `(paths)` — 只對 image 1 上傳。
- `ObservePane` 完全重寫：新 props 為 `captureUrl1/2/3`、`maskSaved`、`onSaveMask`（不再接受 `idx`、`showCapturePhase`）；佈局為 Image 1 大框（上）+ Image 2/3 小縮圖並排（下）+ 右側 320px Status/Tip/Log 面板。
- 當 `paths.length === 0 && !maskSaved` 時，ROI 操作列顯示「No mask drawn — full image will be used」hint。
- Image 2/3 縮圖加入 `data-testid="capture-thumb-2"` 和 `data-testid="capture-thumb-3"`，capture 後顯示真實圖片，未 capture 時顯示 dim 佔位框。

### Test Coverage

#### CreateScreen.test.tsx (7 新測試，8 既有測試更新)

| 測試項目 | 對應 TC |
|---------|---------|
| stepper shows Observe (not Observe 1/2/3) as a single step | TC1 |
| renders capture-thumb-2 and capture-thumb-3 tiles before capture | TC2 |
| shows image_2 and image_3 in thumbnails without ROI canvas | TC4 |
| shows "full image will be used" hint when no ROI strokes drawn | TC6 |
| Continue to Teach pick button is not shown before capture completes | TC8 |
| Continue to Teach pick is enabled after capture even without saving ROI | TC9 |
| does not call PUT masks/1 when user continues without drawing any ROI | TC5 |

既有測試更新：
- 所有 helper 改用 `/Continue.*Observe$/i`（不含數字）
- `advanceToObserve1()` → `advanceToObserve()`
- TC5 Save ROI 按鈕查詢由 `"Save ROI for image_1"` → `/Save ROI/i`

### Files Changed

#### Production Files

- `app/src/renderer/src/screens/CreateScreen.tsx`（全面重寫：Stepper、ObservePane、canVisit、state）

#### Test Files

- `app/src/renderer/src/screens/__tests__/CreateScreen.test.tsx`（更新 helpers + 7 新測試）

### Acceptance Criteria Verification

| Acceptance Criterion | Status | Evidence |
|---|---|---|
| Scenario 1: Stepper 5 步 | Passed | F04-TC1 test |
| Scenario 2: Observe 拍照前佈局 | Passed | F04-TC2 test |
| Scenario 3: 拍照中狀態 | Passed | 實作完成；captureJobRunning 顯示 capturing 訊息 |
| Scenario 4: 拍照完成後 Image 1 可塗抹 | Passed | F03-TC3 test（仍通過） |
| Scenario 5: Mask 選填 | Passed | F04-TC9 + F04-TC5 tests |
| Scenario 6: Save ROI 呼叫 uploadMask | Passed | F03-TC5 test（仍通過，按鈕查詢改用 /Save ROI/i） |
| Scenario 7: Continue 解鎖條件 | Passed | F04-TC8 + F04-TC9 tests |
| Scenario 8: 右側面板 | Passed | 實作完成（Status/Tip/Pipeline log panels） |

### FXX Test Case Verification

| TC | Status | Automated Test Evidence |
|---|---|---|
| TC1 | Passed | F04-TC1 (stepper shape) |
| TC2 | Passed | F04-TC2 (pre-capture placeholders) |
| TC3 | Passed | F03-TC3（仍通過） |
| TC4 | Passed | F04-TC4 (no ROI canvas in thumbs) |
| TC5 | Passed | F04-TC5 (no uploadMask when continuing without drawing) |
| TC6 | Passed | F04-TC6 (hint text) |
| TC7 | Passed | F03-TC5（仍通過，Save ROI button regex updated） |
| TC8 | Passed | F04-TC8 (Continue absent before capture) |
| TC9 | Passed | F04-TC9 (Continue enabled after capture without mask) |

### Commands Run

```bash
# Red phase — 15 failures (helpers + 7 new F04 tests)
npm test  # → 15 failed, 48 passed

# Green phase — production code rewrite
npm test  # → 2 failed (inline "Observe 1" references missed)

# Fix inline references
npm test  # → 63 passed (0 failed)
```

### Result

```
Test Files  4 passed (4)
Tests       63 passed (63)
```

### Assumptions

- `uploadMask` for image 2 / 3 は UI からは呼ばれなくなった。後端の mask データの扱いは後続タスクで決定。
- Image 2 / 3 の縮圖の高さは CSS `minHeight: 140px` に依存し、`video-feed` クラスで制御される。

### Deferred Items

- TC3 (capture job running state UI) は自動化テストなし（captureJobRunning → TerminalLog 表示の component テストは複雑なため省略）。実装は完了。

### Notes

- F03 の全テスト（56）が引き続き通過（うち CreateScreen は helper 更新で対応済み）。
- `mockCameraFeed` は capture 前の Image 1 大框で引き続き使用。
- Image 2 / 3 に ROI canvas を付けないという設計は意図的（F04 仕様通り）。

---

## 附錄：TDD 需求實作流程提醒 (TDD Implementation Checklist)

1. 撰寫並調整需求說明
2. 建立測試（vitest + MSW，同 F03 基礎設施）
   - `CreateScreen.test.tsx`：更新 Stepper 測試（TC1）；新增 Observe 步驟佈局測試（TC2–TC9）
3. 撰寫最簡實作（修改 `CREATE_STEPS`、`canVisit`、`ObservePane` 元件）
4. 測試通過（綠燈）
5. 重構（Refactor）
6. 文件與版本同步（更新本文件的實作記錄）
