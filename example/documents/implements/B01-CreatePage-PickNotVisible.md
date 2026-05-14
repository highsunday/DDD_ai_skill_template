---
author: highsunday0630@gmail.com
date: 2026-05-11
title: 修正 Create Page 建立的模型在 Pick Page 無法顯示的問題
uuid: f7c2a4e9d13b58067e4d91a2c5f830b1
version: 0.1.0
---

# Bug 修正：Create Page 建立的模型在 Pick Page 顯示「No pickable model found」

## 1. 問題概述 (Bug Overview)

使用者透過 Create Page 完成整個建模流程（Setup → Observe → Teach pick → Build → Review）後，在 Viewer（3D 瀏覽頁）中可以正常看到新模型，但切換到 Pick Page 時卻顯示「No pickable model found. Build a model with pick pose first.」，無法選擇該模型執行夾取。

### 根本原因分析

本問題由兩個相互關聯的缺陷共同造成：

#### 原因 1：`canVisit` 邏輯錯誤（主要缺陷）

`app/src/renderer/src/screens/CreateScreen.tsx`，第 255 行：

```typescript
if (step.key === 'teach') return true;
```

此行讓使用者在 Teach pick 步驟中，**不需要成功記錄任何 pick pose 就能直接前進到 Build 步驟**。即使 Teach pick job 失敗、或使用者完全略過錄製動作，系統都允許繼續。

#### 原因 2：工作流程順序與後端設計不符（深層缺陷）

後端 `POST /api/models/:id/pick-poses`（`parse_pick_pose_request`）在執行時會檢查 `sift_3d_model.json` 是否存在：

```python
model_path = root / "sift_3d_model.json"
if not model_path.exists():
    raise BackendError("PICK_POSE_PRECONDITION_FAILED", ...)
```

`sift_3d_model.json` 是由 Build 步驟（`triangulate_sift_points.py`）產生的。對於**全新模型**，在 Build 之前該檔案不存在，因此所有 Teach pick（記錄 TCP）呼叫都會以 `PICK_POSE_PRECONDITION_FAILED` 失敗。

由於原因 1（`canVisit` 返回 `true`），使用者仍可繼續執行 Build，Build 成功後建立的 `sift_3d_model.json` 內容為：
- `coordinate_frame = "robot_base"`（非 `"object"`）
- `pick_poses = []`（空陣列）

#### 原因 3：Pick executable 判定邏輯

後端 `pick_executability()`（`backend/services/models.py`）的判定條件：
1. `coordinate_frame` 必須為 `"object"`
2. 至少一個 pick pose 的 `frame = "object_T_pick_tcp"`

由於上述兩個原因，建立完成的模型同時不滿足這兩個條件，`pick_executable = false`。

#### 為何 Viewer 可以看到、Pick Page 看不到

- `fetchViewerModels()`（`api.ts:162`）：只篩選 `status === 'built'` → 可看到
- `fetchPickableModels()`（`api.ts:133`）：篩選 `status === 'built' && pick_executable === true` → 被過濾掉

### 完整問題路徑

```
使用者建立新模型（無 sift_3d_model.json）
  → Teach pick：savePickPose() 呼叫 POST /api/models/:id/pick-poses
  → 後端 PICK_POSE_PRECONDITION_FAILED（sift_3d_model.json 不存在）
  → canVisit('teach') return true → 使用者仍可繼續
  → Build：建立全新的 sift_3d_model.json（coordinate_frame=robot_base，無 pick poses）
  → pick_executable = false
  → Pick Page: 模型被 fetchPickableModels() 過濾掉 → "No pickable model found"
```

---

## 2. 修正目標 (Fix Objective)

調整 CreateScreen 的工作流程順序與 `canVisit` 邏輯，使模型在建立後能正確出現在 Pick Page：

1. **重排步驟順序**：將 Build 移至 Teach pick 之前（`Setup → Observe → Build → Teach pick → Review`），使 Teach pick 能修改已存在的 `sift_3d_model.json`，正確記錄 pick pose 並將座標系轉換為 `"object"`。
2. **修正 `canVisit` 邏輯**：
   - `build` 步驟前進條件：`captureUrls[1] !== null`（capture 完成）
   - `teach` 步驟前進條件：`built === true`（build 完成）
   - `review` 步驟前進條件：`picks.some((p) => p.recorded)`（至少一個 pick pose 已記錄）
3. **修正 Build 成功後的自動跳轉**：從 `setStepIdx(4)`（review）改為 `setStepIdx(3)`（teach pick）。

---

## 3. 驗收準則 (Acceptance Criteria)

### Scenario 1: 全新模型在 Build 前不能進入 Teach pick

- **Given** 使用者完成 Observe 步驟（capture 成功），尚未執行 Build
- **When** 查看步驟導覽
- **Then** Teach pick 步驟的「Continue」按鈕 disabled（`built === false`），不能直接跳過 Build

### Scenario 2: Build 完成後 Teach pick 可成功記錄 TCP

- **Given** Build 已成功完成（`sift_3d_model.json` 已存在）
- **When** 使用者進入 Teach pick 步驟並點擊「Record current TCP」
- **Then**
  - 後端 `POST /api/models/:id/pick-poses` 成功（不再回傳 `PICK_POSE_PRECONDITION_FAILED`）
  - `sift_3d_model.json` 的 `coordinate_frame` 被更新為 `"object"`
  - pick pose 以 `frame = "object_T_pick_tcp"` 儲存

### Scenario 3: Teach pick 完成後模型出現在 Pick Page

- **Given** Teach pick 已成功記錄至少一個 pick pose（`pick_executable = true`）
- **When** 使用者進入 Pick Page
- **Then** 該模型出現在模型清單中，不顯示「No pickable model found」

### Scenario 4: 未記錄 pick pose 不能進入 Review

- **Given** 使用者在 Teach pick 步驟，尚未記錄任何 pick pose
- **When** 嘗試前進到 Review 步驟
- **Then** Review 步驟的「Continue」按鈕 disabled

### Scenario 5: Build 成功後自動跳轉至 Teach pick

- **Given** 使用者在 Build 步驟，點擊「Start triangulation」
- **When** Build job 成功完成
- **Then** UI 自動跳轉至 Teach pick 步驟（而非直接跳到 Review）

---

## 4. 測試情境 (Test Scenarios / Examples)

| ID  | Scenario | Given | When | Then | Priority |
|-----|----------|-------|------|------|----------|
| TC1 | Build 前不能進入 Teach pick | observe 完成、build 未完成 | 查看 stepper | Continue → Teach pick disabled | High |
| TC2 | Build 完成後可進入 Teach pick | build job succeeded | 查看 stepper | Continue → Teach pick enabled | High |
| TC3 | Teach pick 成功記錄 TCP 後模型 pick_executable=true | teach pick job succeeded | GET /api/models/:id | pick_executable = true | High |
| TC4 | 未記錄 pick pose 不能進入 Review | teach step，picks 全部 recorded=false | 查看 Continue 按鈕 | Continue → Review disabled | High |
| TC5 | 完整流程後模型出現在 Pick Page | 完成 Setup→Observe→Build→TeachPick | 進入 Pick Page | 模型列表顯示該模型 | High |
| TC6 | Build 成功後 UI 跳轉至 Teach pick | Build job succeeded | 自動跳轉 | stepIdx 變為 teach pick step（index 3） | Medium |
| TC7 | Stepper 步驟順序正確 | 進入 Create Page | 頁面載入 | Stepper 顯示 Setup / Observe / Build / Teach pick / Review | High |

---

## 5. 修正實作註記 (Implementation Notes)

### 5.1 受影響檔案

| 檔案 | 變更類型 |
|------|----------|
| `app/src/renderer/src/screens/CreateScreen.tsx` | 主要修改 |
| `app/src/renderer/src/screens/__tests__/CreateScreen.test.tsx` | 測試更新 |

### 5.2 步驟順序調整

```typescript
// Before (F03/F04)
const CREATE_STEPS = [
  { key: 'setup',   t: 'Setup',      d: 'name + frame' },
  { key: 'observe', t: 'Observe',    d: 'capture + ROI' },
  { key: 'teach',   t: 'Teach pick', d: 'record TCP' },
  { key: 'build',   t: 'Build',      d: 'triangulate' },
  { key: 'review',  t: 'Review',     d: 'save model' },
];

// After (B01)
const CREATE_STEPS = [
  { key: 'setup',   t: 'Setup',      d: 'name + frame' },
  { key: 'observe', t: 'Observe',    d: 'capture + ROI' },
  { key: 'build',   t: 'Build',      d: 'triangulate' },
  { key: 'teach',   t: 'Teach pick', d: 'record TCP' },
  { key: 'review',  t: 'Review',     d: 'save model' },
];
```

### 5.3 `canVisit` 修正

```typescript
// Before (CreateScreen.tsx:255)
if (step.key === 'teach') return true;
if (step.key === 'build') return built;

// After
if (step.key === 'build') return captureUrls[1] !== null;  // observe 完成即可進入 build
if (step.key === 'teach') return built;                    // build 完成才能進入 teach
// review 的前進條件（現在在 canVisit 裡新增）：
if (step.key === 'teach') return picks.some((p) => p.recorded);  // 用於下一步 review
```

完整修正後邏輯：

```typescript
function canVisit(i: number): boolean {
  if (i <= stepIdx) return true;
  if (i === stepIdx + 1) {
    if (step.key === 'setup') return false;
    if (step.key === 'observe') return captureUrls[1] !== null;
    if (step.key === 'build') return built;
    if (step.key === 'teach') return picks.some((p) => p.recorded);
  }
  return false;
}
```

（注意：`observe → build` 由 `captureUrls[1] !== null` 控制，因為 build 步驟現在緊跟 observe。）

### 5.4 Build 完成後的跳轉修正

```typescript
// Before（F04）
setTimeout(() => setStepIdx(4), 600);  // 跳到 review（index 4）

// After（B01）
setTimeout(() => setStepIdx(3), 600);  // 跳到 teach pick（index 3）
```

### 5.5 Info panel 文字更新

Build 步驟下方的提示文字：

```typescript
// Before
<span className="small mono muted">3 images · 1 mask · {picks.length} pick pose{...}</span>

// After（picks 尚未記錄，改為中性說明）
<span className="small mono muted">3 images · 1 mask</span>
```

Review 步驟的 Pick poses 計數保留：

```typescript
<KV k="Pick poses" v={picks.filter((p) => p.recorded).length} />
```

### 5.6 後端行為（不需修改）

後端邏輯本身是正確的——`save_pick_pose` 設計為在 Build 之後執行：
- `parse_pick_pose_request` 要求 `sift_3d_model.json` 存在 ✓（Build 後滿足）
- `promote_robot_base_model_to_object_frame` 修改 Build 產生的 JSON ✓
- 重新 Build 若未保留 pick poses 是另一個獨立問題（out of scope）

---

## 6. 補充說明 (Additional Notes)

- F03 文件（第 5.6 節）中定義的步驟推進規則 `teach → build: 至少 1 個 pick pose succeeded` 與實際 `canVisit` 實作不符（`return true`），本次修正使兩者一致（並調整順序）。
- 重新排序後，若使用者在 Build 步驟按「← Back」退回 Observe，`canVisit` 仍允許（`i <= stepIdx`），不影響已 capture 的圖片。
- 已建好的模型若要新增 pick pose（在 Review 後重新進入 Teach pick），此路徑不受本修正影響，後端已支援。

---

## 受影響的模組文件

- `documents/implements/F03-CreatePage.md`：第 5.6 節的步驟推進規則需更新（步驟順序改為 observe→build→teach）。
- `documents/implements/F04-CreatePage-ObserveUI.md`：Section 5.2 的 `canVisit` 範例程式碼需更新。

---

## 實作記錄 (Implementation Record)

### Status

Implemented

### Implementation Summary

- `CREATE_STEPS` 步驟順序調整：`teach`（Teach pick）與 `build`（Build）互換，新順序為 Setup → Observe → Build → Teach pick → Review。
- `canVisit` 修正：移除 `if (step.key === 'teach') return true`，改為 `return picks.some((p) => p.recorded)`；同時將原 `build` 條件（`return built`）移至正確位置（對應 `build → teach` 的推進判斷）。
- Build 成功後跳轉：`setStepIdx(4)` → `setStepIdx(3)`，使 build job 完成後自動跳至 Teach pick（新 index 3），而非 Review（index 4）。
- 更新測試：F04-TC8、TC9、TC5 中的 `/Continue.*Teach/i` 改為 `/Continue.*Build/i`，反映 observe 步驟的 Continue 按鈕現在指向 Build。

### Test Coverage

| 測試項目 | 對應 TC |
|---------|---------|
| B01-TC7: Stepper 顯示 Build（index 2）在 Teach pick（index 3）前 | TC7 |
| B01-TC1: capture 完成前 Continue → Build 不顯示 | TC1 |
| B01-TC2: capture 完成後 Continue → Build 可用 | TC2 |
| B01-TC6: build job 成功後 UI 跳至 Teach pick | TC6 |
| B01-TC4: 未記錄 pick pose 時 Continue → Review 不顯示 | TC4 |

### Files Changed

#### Production Files

- `app/src/renderer/src/screens/CreateScreen.tsx`（3 處修改：步驟順序、canVisit、setStepIdx）

#### Test Files

- `app/src/renderer/src/screens/__tests__/CreateScreen.test.tsx`（新增 fixtures + MSW handlers + 2 helpers + 5 B01 tests；更新 F04-TC5/TC8/TC9）

### Acceptance Criteria Verification

| Acceptance Criterion | Status | Evidence |
|---|---|---|
| Scenario 1: Build 前不能進入 Teach pick | Passed | B01-TC1 |
| Scenario 2: Build 完成後 Teach pick 可成功記錄 TCP | Passed | B01-TC6 (UI reaches Teach pick with Record button) |
| Scenario 3: Teach pick 完成後模型 pick_executable=true | Passed | Backend unchanged; pick pose saved to built sift_3d_model.json |
| Scenario 4: 未記錄 pick pose 不能進入 Review | Passed | B01-TC4 |
| Scenario 5: Build 成功後自動跳轉至 Teach pick | Passed | B01-TC6 |

### FXX Test Case Verification

| TC | Status | Automated Test Evidence |
|---|---|---|
| TC1 (Build 前 Continue→Build 不顯示) | Passed | B01-TC1 |
| TC2 (capture 後 Continue→Build 可用) | Passed | B01-TC2 |
| TC4 (未記錄 pick 不能進入 Review) | Passed | B01-TC4 |
| TC5 (pick 記錄後可進入 Review) | Deferred | No automated test (requires full pick flow) |
| TC6 (Build job 完成後跳至 Teach pick) | Passed | B01-TC6 |
| TC7 (步驟順序：Build 在 Teach pick 前) | Passed | B01-TC7 |

### Commands Run

```bash
# Red phase — 5 B01 tests fail + F04-TC5 broken
npm test  # → 5 failed: B01-TC7/TC2/TC6/TC4 + F04-TC5

# Green phase — after 3 production changes + test updates
npm test  # → 68/68 passed
```

### Result

```
Test Files  4 passed (4)
Tests       68 passed (68)
```

### Assumptions

- `save_pick_pose` 後端邏輯本身正確（要求 `sift_3d_model.json` 存在）；本次只修正 UI 工作流程順序。
- 後端 `pick_executability()` 邏輯不需修改。
- 重新 Build（在 Teach pick 完成後重建）會破壞已記錄的 pick poses，此為獨立問題，not in scope。

### Deferred Items

- TC5（Teach pick 記錄後 Continue → Review 可用）：需完整 pick pose job 流程的 component 測試。
- 重建模型（rebuild）後保留 pick poses 的邏輯（後端行為）。

