---
author: highsunday0630@gmail.com
date: 2026-05-12
title: Pick Page — Detect tab TCP selector without image SVG overlay
uuid: 9b4e2c7f1d3a58e046b7c9f2e1d4a083
version: 0.1.0
---

# 功能需求書 - Pick Page Detect 頁簽 TCP 選擇器與無疊圖結果圖

## 1. 功能概述 (Feature Overview)

目前 Pick Page 的夾取位置（pick pose / TCP）必須在 **Select 步驟**選好，才能啟動 workflow。這使得使用者若想對同一個模型嘗試不同 TCP，每次都要回到 Select step，操作動線長、實驗效率低。

本功能有兩個子目標：

**子目標 A — Detect tab TCP 選擇器**  
將 TCP 選擇移入 **Detect 步驟**，Select 步驟只需選模型，TCP 自動帶入第一個可用值。  
在 Detect 步驟新增 TCP 下拉選單；使用者切換 TCP 後可一鍵重新偵測（re-detect），不需重新走 Select。  
這讓使用者對同一顆模型快速切換不同抓取位置，縮短 test cycle。

**子目標 B — Detect 結果圖保持純影像**  
Detect 步驟的結果圖（帶 SIFT 特徵的偵測圖）必須保持後端回傳影像原貌。  
前端不得在結果圖上疊加 SVG、canvas、marker、label、tooltip target 或任何 TCP 視覺標記；TCP 選擇只透過 Detect 步驟的 TCP selector 操作。

---

## 2. 需求描述 (Requirement / User Story)

- **As a** 研發工程師（demo 操作者）
- **I want** 在 Detect 步驟直接切換 TCP 並重新偵測，且結果圖不要被前端 SVG 或標記覆蓋
- **So that** 對同一模型測試不同夾取位置時，不需要每次回到 Select 步驟，加快 test cycle

---

## 3. 驗收準則 (Acceptance Criteria)

### Scenario 1: Select 步驟不需選 TCP
- **Given** 使用者進入 Pick Page，模型清單載入完成
- **When** Select 步驟顯示
- **Then**
  - 「Choose model」Panel 仍顯示模型清單
  - **不顯示 Pick pose 下拉選單**（已移除，改到 Detect 步驟）
  - 使用者只需選模型，按 Continue 即可進入 Observe
  - `pickName` 自動設為該模型第一個可用 TCP（`pick_poses[0].name`）

### Scenario 2: Detect 步驟顯示 TCP 選擇器
- **Given** Dry-run job 成功，UI 進入 Detect 步驟
- **When** Detect step 渲染
- **Then**
  - 顯示 TCP 下拉選單，列出目前選擇模型的所有 `pick_poses`
  - 下拉選單預設選取目前已偵測的 TCP（`dryRunResult.estimated_pick_pose.name`）
  - 選單旁有「Re-detect」按鈕

### Scenario 3: 切換 TCP 並重新偵測
- **Given** Detect 步驟顯示偵測結果，使用者從 TCP 下拉選單選了另一個 TCP
- **When** 使用者按下「Re-detect」按鈕
- **Then**
  - 清空現有 `dryRunResult`、重設 robot log
  - 以新的 `pick_name` 重新呼叫 `POST /api/pick/workflow`（`dry_run=true`）
  - Job polling 繼續，完成後以新結果更新 Detect 步驟 UI
  - **不**回到 Select / Observe 步驟（UI 停留在 Detect）

### Scenario 4: Detect 結果圖不顯示任何前端 SVG 疊圖
- **Given** Dry-run job 成功，`result_image_url` 有效
- **When** Detect 步驟顯示結果圖
- **Then**
  - 結果圖以單一 `<img>` 顯示 `result_image_url`
  - 圖片容器內不得出現 `<svg>`、`canvas`、TCP marker、TCP label 或任何前端繪製的 overlay
  - 不顯示 `data-testid="tcp-overlay"` 或 `data-testid` 以 `tcp-marker-` 開頭的元素
  - TCP name 與 `object_pick_tcp_pose` 可在圖片外的文字區塊顯示，但不得覆蓋在圖片上

### Scenario 5: 模型只有一個 TCP 時的降級行為
- **Given** 模型只有一個 pick pose
- **When** Detect 步驟顯示
- **Then**
  - TCP 選擇器改為純顯示（disabled 狀態），不提供下拉選擇
  - 結果圖仍不顯示任何 TCP 標記或 SVG overlay
  - UI 行為與原本 Detect step 一致

### Scenario 6: Re-detect 時 Observe 仍會重新執行
- **Given** 使用者在 Detect 步驟切換 TCP 並按 Re-detect
- **When** 新 dry_run job 執行
- **Then**
  - Job 日誌清空後重新累積（機械臂會再次移動並拍照）
  - 完成後更新 Detect 步驟所有指標（matches、inliers、pose）
  - 若 job 失敗，顯示錯誤並維持 Detect 步驟；不退回 Select

---

## 4. 測試情境 (Test Scenarios / Examples)

| ID  | Scenario                                      | Given                                    | When                              | Then                                                              | Priority |
|-----|-----------------------------------------------|------------------------------------------|-----------------------------------|-------------------------------------------------------------------|----------|
| TC1 | Select 步驟無 TCP 下拉                        | 模型清單載入完成（2 個 TCP 模型）        | 進入 Select step                  | 頁面無 Pick pose 下拉選單；TCP 自動設為第一個                     | High     |
| TC2 | Detect 步驟顯示 TCP selector                  | Dry-run 成功                             | 進入 Detect step                  | TCP 下拉列出所有 pick_poses；預設值 = 偵測結果 TCP                | High     |
| TC3 | 切換 TCP 並 Re-detect 成功                    | Detect step 有 2 個 TCP；切換到第 2 個   | 按 Re-detect                      | 重新執行 dry_run with pick_name = tcp2；完成後結果更新             | High     |
| TC4 | Re-detect 停留在 Detect step                  | 切換 TCP 後按 Re-detect                  | Job 執行中                        | stepIdx 不變；Observe/Select 步驟不跳轉                           | High     |
| TC5 | 單 TCP 模型 selector 不可選                   | 模型只有 1 個 pick pose                  | Detect step 顯示                  | TCP selector 為 disabled；Re-detect 按鈕仍可用（重跑同一 TCP）    | Medium   |
| TC6 | Detect 結果圖無 SVG overlay                   | 模型有 3 個 pick poses；偵測成功         | Detect step 顯示 result_image_url | 圖片容器只有結果 `<img>`；沒有 `<svg>`、canvas、marker 或 label   | High     |
| TC7 | TCP 資訊不覆蓋在結果圖上                      | 多 TCP 模型偵測成功                      | Detect step 顯示                  | TCP selector 可切換；若顯示 TCP pose 資訊，必須位於圖片外         | Medium   |
| TC8 | Re-detect 失敗後維持 Detect step              | 切換 TCP，Re-detect；後端 job 失敗       | Job status=failed                 | 錯誤訊息在 log 與 banner 顯示；TCP selector 可再次操作            | Medium   |
| TC9 | Re-detect 時 Log 正確重置                     | 第一次 dry_run 已有 logs                 | 按 Re-detect                      | 舊 logs 清空；新 logs 從 0 累積                                   | Medium   |

---

## 5. 實作註記 (Implementation Notes)

### 5.1 Select 步驟修改

- 移除 `<Panel title="Pick + motion parameters">` 裡的 Pick pose `<select>` 元件。
- Motion parameters（Speed、Accel、Approach、Observe position）保留於 Select 步驟，或可一併移至 Detect 步驟（Out of scope，實作時確認）。
- `pickName` state 初始化仍從 `model.pick_poses[0].name` 帶入；Select step 不需讓使用者操作。

### 5.2 Detect 步驟新增 TCP selector

在 Detect step 的 Detection result Panel（或 Pose output Panel）加入：

```tsx
<label className="field">TCP（pick pose）
  <select
    value={pickName}
    onChange={(e) => setPickName(e.target.value)}
    disabled={model.pick_poses.length <= 1}
  >
    {model.pick_poses.map((p) => <option key={p.name} value={p.name}>{p.name}</option>)}
  </select>
</label>
<button
  className="btn ghost"
  disabled={isJobRunning}
  onClick={reDetect}
>
  Re-detect
</button>
```

`reDetect()` 函式：

```ts
async function reDetect() {
  setDryRunResult(null);
  setJobError(null);
  setRobotLog([]);
  lastLogCount.current = 0;
  jobPhaseRef.current = 'dry_run';
  pushLog(`Re-detecting with TCP: ${pickName} …`, 'info');
  try {
    const { job_id } = await startPickWorkflow({
      model_id: modelId,
      pick_name: pickName,
      observe_name: 'observe',
      approach_height_m: approachHeight,
      dry_run: true,
    });
    setActiveJobId(job_id);
  } catch (e) { /* same as startObserve error handling */ }
}
```

> 注意：`reDetect` 與 `startObserve` 邏輯幾乎相同，可以合併成一個函式並以 `keepStep` flag 區分是否推進步驟。

### 5.3 Detect 結果圖渲染限制

結果圖必須使用後端回傳的 `result_image_url` 直接渲染為 `<img>`。前端不得在圖片上建立 SVG overlay、canvas overlay、絕對定位 marker、文字 label 或 hover target。

允許：
- Detect step 的 TCP selector 顯示目前可用 TCP。
- Re-detect 按鈕依目前 selector 值重新送出 workflow。
- TCP name 或 `object_pick_tcp_pose` 作為圖片外的文字資訊顯示。

禁止：
- 在結果圖容器內加入 `<svg>` 或 `<canvas>`。
- 使用 `position: absolute` 將 TCP marker、label、tooltip target 疊在圖片上。
- 產生 `data-testid="tcp-overlay"` 或 `data-testid` 以 `tcp-marker-` 開頭的元素。
- 為了 TCP 標記而新增前端 2D 投影計算。

### 5.4 `PickWorkflowResult` 型別註記

本需求不再要求為前端圖片 overlay 新增投影資料。若 `app/src/renderer/src/lib/api.ts` 的 `PickWorkflowResult` 已有 `estimated_object_pose`，可保留供偵測結果或除錯資訊使用；若尚未存在，不需為 F05 的無 overlay 行為新增。

```ts
estimated_object_pose: {
  frame: string;
  pose: [number, number, number, number, number, number];
  matrix: number[][];
} | null;
```

### 5.5 stepIdx 行為

- `reDetect()` 呼叫後 **不改變 `stepIdx`**（維持在 detect = 2）
- Polling 成功後只更新 `dryRunResult`，不推進 `stepIdx`
- Detect step 中 `setDryRunResult(null)` 時，結果區塊顯示 loading 或空狀態

---

## 6. 受影響的模組與檔案

| 檔案 | 變更類型 |
|---|---|
| `app/src/renderer/src/screens/PickScreen.tsx` | 主要修改：Select 移除 TCP picker；Detect 新增 TCP selector + Re-detect；結果圖不得有 SVG/canvas overlay |
| `app/src/renderer/src/lib/api.ts` | 保持 `PickWorkflowResult` 與 re-detect 需求相容；不得為前端 overlay 額外要求投影欄位 |
| `app/src/renderer/src/screens/__tests__/PickScreen.test.tsx` | 新增 TC1–TC9 對應測試 |

> 本版明確不做圖片上的 TCP 投影或 SVG 疊圖，因此不需要新增 `mathUtils.ts` 或前端矩陣投影工具。

---

## 7. 開放問題 (Open Questions)

| # | 問題 | 影響 |
|---|---|---|
| Q1 | 若未來需要視覺化 TCP 位置，是否改在圖片外以表格/列表呈現？ | 避免再次把 overlay 疊回結果圖 |
| Q2 | 後端 `PickWorkflowResult` 目前是否已回傳 `estimated_object_pose`？ | 只作資料顯示用途；不得用於前端圖片 overlay |
| Q3 | Select 步驟的 Motion parameters（Speed/Accel/Approach）是否一併移至 Detect 步驟？ | UI 動線設計 |
| Q4 | Re-detect 時是否需要重新移動機械臂並拍照（目前無「只重算 pose」的後端 API）？ | 若後端能支援 "re-estimate from existing capture" 效率更高 |

---

## 8. Out of Scope

- Select 步驟 Motion parameters 移位（Q3，另評估）
- 後端新增「從已拍圖重算 pose」API（Q4，若有需求另開 FXX）
- Detect 結果圖上的 SVG/canvas/TCP marker overlay
- Execute workflow 中途切換 TCP
- Viewer 或 Create page 的 TCP 相關異動

---

## 實作記錄 (Implementation Record)

### Status

Implemented; updated on 2026-05-13 to remove the image SVG overlay from code and tests.

### Implementation Summary

- **Select step**: 移除 Pick pose `<select>` 下拉選單，Panel 標題從「Pick + motion parameters」改為「Motion parameters」。`pickName` 仍自動初始化為 `model.pick_poses[0].name`。
- **Detect step**:
  - 新增 `data-testid="tcp-selector"` 的 TCP 選擇器（多 TCP 時可選，單 TCP 時 `disabled`）。
  - 新增「Re-detect」按鈕，呼叫 `reDetect()` 函式。
  - 結果圖改為只渲染後端 `result_image_url` 的 `<img>`；已移除 `tcp-overlay` SVG 與 `tcp-marker-*` TCP marker。
- **`reDetect()` 函式**:
  - 清空 `dryRunResult`、`jobError`、`robotLog`
  - 設定 `jobPhaseRef.current = 're_detect'`（新增）
  - 立即呼叫 `pushLog('Re-detecting with TCP: ${pickName} …')` 再觸發 POST
- **Job polling 更新**: 新增 `'re_detect'` phase 判斷；`re_detect` 成功時只更新 `dryRunResult`，不改變 `stepIdx`（停留在 Detect step）。
- **`api.ts`**: `PickWorkflowResult` 可保留 `estimated_object_pose` 欄位（`| null`），但不得用於在結果圖上渲染 TCP overlay。
- **2026-05-13 決策**: 不採用任何圖片 overlay 方案；結果圖保持單純 `<img>`，TCP 選擇只透過 Detect step selector。

### Test Coverage

| 測試 | 對應 TC |
|---|---|
| `F05-TC1: does not show a Pick pose dropdown in the Select step` | TC1 |
| `F05-TC1: first pick pose is auto-selected` | TC1 |
| `F05-TC2: shows TCP selector with all pick poses listed after dry-run succeeds` | TC2 |
| `F05-TC2: TCP selector default value matches the detected TCP` | TC2 |
| `F05-TC2: Re-detect button is present in Detect step` | TC2 |
| `F05-TC3: sends pick_name of newly selected TCP in the Re-detect POST` | TC3 |
| `F05-TC4: remains on Detection result panel after Re-detect completes` | TC4 |
| `F05-TC5: TCP selector is disabled when model has only one pick pose` | TC5 |
| `F05-TC5: Re-detect button is still enabled for single-TCP model` | TC5 |
| `F05-TC6: does not render a TCP overlay element in Detect step` | TC6 |
| `F05-TC6: does not render TCP marker elements over the result image` | TC6 |
| `F05-TC6: keeps the result image container free of svg and canvas elements` | TC6 |
| `F05-TC7: keeps TCP names out of the result image container` | TC7 |
| `F05-TC7: TCP pose information, if shown, is outside the result image` | TC7 |
| `F05-TC8: shows error in log and stays on Detect step when Re-detect job fails` | TC8 |
| `F05-TC9: resets logs when Re-detect is triggered (pushes new Re-detecting log entry)` | TC9 |

### Files Changed

#### Production Files

- `app/src/renderer/src/screens/PickScreen.tsx`（移除 Detect 結果圖上的 SVG overlay / TCP markers；保留結果 `<img>`、TCP selector 與 Re-detect 行為）

#### Test Files

- `app/src/renderer/src/screens/__tests__/PickScreen.test.tsx`（TC6/TC7 修訂為驗證無 SVG/canvas/marker overlay，並確認 TCP 選擇仍由 selector 提供）

### Acceptance Criteria Verification

| Acceptance Criterion | Status | Evidence |
|---|---|---|
| Scenario 1: Select 步驟不需選 TCP | Passed | F05-TC1 兩個測試 |
| Scenario 2: Detect 步驟顯示 TCP selector | Passed | F05-TC2 三個測試 |
| Scenario 3: 切換 TCP 並重新偵測 | Passed | F05-TC3, TC4 |
| Scenario 4: Detect 結果圖不顯示任何前端 SVG 疊圖 | Passed | F05-TC6, TC7 |
| Scenario 5: 單 TCP 模型 selector disabled | Passed | F05-TC5 |
| Scenario 6: Re-detect 時 Observe 重新執行 | Passed | F05-TC3 (pick_name 正確傳送), TC8 (failure path) |

### FXX Test Case Verification

| TC | Status | Automated Test Evidence |
|---|---|---|
| TC1 | Passed | `F05-TC1` (2 tests) |
| TC2 | Passed | `F05-TC2` (3 tests) |
| TC3 | Passed | `F05-TC3` |
| TC4 | Passed | `F05-TC4` |
| TC5 | Passed | `F05-TC5` (2 tests) |
| TC6 | Passed | `F05-TC6` (3 tests) |
| TC7 | Passed | `F05-TC7` (2 tests) |
| TC8 | Passed | `F05-TC8` |
| TC9 | Passed | `F05-TC9` |

### Commands Run

```bash
# Red phase for 2026-05-13 no-overlay update
npm test -- PickScreen.test.tsx --reporter=verbose -t "F05-TC6|F05-TC7"
# 4 failed, 1 passed: existing SVG `tcp-overlay` and `tcp-marker-*` were still rendered

# Green phase
npm test -- PickScreen.test.tsx --reporter=verbose -t "F05-TC6|F05-TC7"
# 5 passed, 19 skipped

# Related screen regression
npm test -- PickScreen.test.tsx --reporter=verbose
# 24 passed

# Full app regression
npm test -- --reporter=verbose
# 83 passed
```

### Result

```
Test Files  4 passed (4)
Tests       83 passed (83)
```

### Assumptions

- 結果圖不得顯示 SVG/canvas/TCP marker overlay；`object_pick_tcp_pose` 不再映射到圖片座標。
- `advanceToDetectWith` helper 不點擊「Move to observe」按鈕，因為「Continue → Observe」已在內部呼叫 `startObserve()`；若重複點擊會產生第二個 POST 使測試邏輯錯位。

### Deferred Items

- 若未來需要 TCP 視覺化，需另開需求並限制在圖片外呈現，或先取得使用者同意再變更本限制。
- Select 步驟 Motion parameters 移位（Q3）。
- 後端「從已拍圖重算 pose」API（Q4）。

---

## 附錄：TDD 需求實作流程提醒 (TDD Implementation Checklist)

1. 撰寫並調整需求說明
2. 建立測試（TC1–TC9）
3. 撰寫最簡實作（先移除 Select TCP picker；再加 Detect TCP selector + Re-detect；最後確認結果圖無 SVG/canvas/TCP marker overlay）
4. 測試通過（綠燈）
5. 重構（如抽出 reDetect 共用邏輯）
6. 文件與版本同步
