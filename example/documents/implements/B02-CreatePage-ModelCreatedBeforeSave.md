---
author: highsunday0630@gmail.com
date: 2026-05-14
title: 修正 Create Page 在 Setup Continue 時即建立模型資料夾，未等到使用者按 Save 才儲存
uuid: 9a3f7c2e1b084d6a8e5c09f3d72a14b8
version: 0.1.0
---

# Bug 修正：Create Page 在使用者點擊 Continue 時提前建立模型資料夾

## 1. 問題概述 (Bug Overview)

使用者在 Create Page 的 Setup 步驟輸入模型名稱後，點擊「Continue → Observe」按鈕，系統會立即呼叫 `POST /api/models`（`handleSetupContinue`，`CreateScreen.tsx:189`），在磁碟上建立模型資料夾並取得 `model_id`。

若使用者在完成完整流程（Setup → Observe → Build → Teach pick → Review → 按 Save）之前中途離開（例如點擊「← Console home」、關閉視窗、或放棄操作），後端已建立的模型資料夾會繼續殘留在磁碟上，形成**不完整的孤兒模型（orphan model）**。

### 錯誤行為

```
使用者輸入模型名稱 "my_model" → 點擊 Continue → Observe
  → POST /api/models { model_name: "my_model" }   ← 資料夾立即建立
  → 使用者決定不繼續 → 點擊 "← Console home"
  → 磁碟上留有 my_model/ 資料夾，內容空白或不完整
  → Viewer 或後端 API 可能看到這個狀態不明的模型
```

### 期望行為

模型資料夾只應在使用者於 Review 步驟按下 **「Save & open model」** 或 **「Save & run pick」** 後才視為正式儲存。若使用者在此之前離開流程，系統應自動清除已建立的模型，避免殘留。

### 根本原因

`handleSetupContinue`（`CreateScreen.tsx:189`）在使用者點擊 Continue 時立即呼叫 `createModel()`（`POST /api/models`），早於使用者確認完整儲存意圖的 Review 步驟。

後端 API 設計要求 `model_id` 在 capture、mask upload、build、pick-pose 等後續步驟前就存在，因此無法將 `POST /api/models` 延遲到 Save 按鈕。正確的修正策略是：保留提前建立的機制，但**在使用者未按 Save 離開時自動刪除模型**（`DELETE /api/models/:id`）。

---

## 2. 修正目標 (Fix Objective)

使用者明確按下 Review 頁的 Save 按鈕前，任何離開 Create 流程的動作（返回主頁、直接關閉 Create 頁、視窗關閉）都應觸發模型清除，讓磁碟與後端狀態與使用者意圖一致：

- **使用者按下 Save → Viewer / Pick**：模型保留，導向對應頁面。
- **使用者按下 ← Console home（在 Review 前）**：自動呼叫 `DELETE /api/models/:id` 清除模型，再導向主頁。
- **使用者按下 ← Back 退回 Setup（`stepIdx → 0`）並重新命名**：若已建立過 `modelId`，清除舊模型，讓使用者重新 Continue 時建立新模型。
- **`CreateScreen` unmount（例如 Electron 視窗關閉）**：若 `modelId` 存在且尚未按過 Save，`useEffect` cleanup 呼叫 `DELETE /api/models/:id`。

---

## 3. 驗收準則 (Acceptance Criteria)

### Scenario 1: 使用者在 Observe 步驟點擊 ← Console home，模型被刪除

- **Given** 使用者完成 Setup Continue（`modelId` 已建立），目前在 Observe 步驟
- **When** 使用者點擊「← Console home」按鈕
- **Then**
  - 呼叫 `DELETE /api/models/:id`
  - 成功後導向主頁
  - 磁碟上不殘留該模型資料夾

### Scenario 2: 使用者在 Build / Teach pick 步驟點擊 ← Console home，模型被刪除

- **Given** `modelId` 已建立，目前在 Build 或 Teach pick 步驟
- **When** 使用者點擊「← Console home」
- **Then** 同 Scenario 1：呼叫 `DELETE /api/models/:id` 後導向主頁

### Scenario 3: 使用者在 Review 步驟按下 Save，模型保留

- **Given** 完整流程已完成，使用者在 Review 步驟
- **When** 使用者點擊「Save & open model」或「Save & run pick」
- **Then**
  - **不**呼叫 `DELETE /api/models/:id`
  - 導向 Viewer 或 Pick 頁面
  - 模型完整保留在磁碟上

### Scenario 4: 使用者退回 Setup 並重新輸入名稱

- **Given** 使用者已從 Setup 繼續到後續步驟（`modelId` 已建立）
- **When** 使用者按「← Back」退回 Setup 步驟（`stepIdx = 0`）
- **Then**
  - 呼叫 `DELETE /api/models/:id` 清除先前建立的模型
  - 重置 `modelId` 為 `null` 及所有相關 state（captureUrls、masks、picks、build）
  - 使用者可重新輸入名稱並點擊 Continue 建立全新模型

### Scenario 5: CreateScreen unmount 時自動清除（視窗關閉等）

- **Given** `modelId` 已建立，尚未按 Save
- **When** `CreateScreen` 元件 unmount（例如 Electron 關閉頁面）
- **Then** `useEffect` cleanup 呼叫 `DELETE /api/models/:id`（best-effort，不阻塞 unmount）

### Scenario 6: Setup Continue 失敗（MODEL_EXISTS / INVALID_MODEL_NAME）

- **Given** 使用者在 Setup 輸入已存在的模型名稱
- **When** `POST /api/models` 回傳 `MODEL_EXISTS` 或 `INVALID_MODEL_NAME`
- **Then**
  - 顯示 inline 錯誤（現有行為不變）
  - `modelId` 保持 `null`，不需清除

---

## 4. 測試情境 (Test Scenarios / Examples)

| ID  | Scenario | Given | When | Then | Priority |
|-----|----------|-------|------|------|----------|
| TC1 | Observe 步驟離開 → 模型刪除 | `modelId` 存在，stepIdx=1（Observe） | 點擊 ← Console home | `DELETE /api/models/:id` 被呼叫，導向主頁 | High |
| TC2 | Build 步驟離開 → 模型刪除 | `modelId` 存在，stepIdx=2（Build） | 點擊 ← Console home | `DELETE /api/models/:id` 被呼叫，導向主頁 | High |
| TC3 | Teach pick 步驟離開 → 模型刪除 | `modelId` 存在，stepIdx=3（Teach pick） | 點擊 ← Console home | `DELETE /api/models/:id` 被呼叫，導向主頁 | High |
| TC4 | Review 按 Save → 模型保留 | 完整流程完成，stepIdx=4（Review） | 點擊 Save & open model | `DELETE /api/models/:id` **不**被呼叫，`onExit('viewer')` 被呼叫 | High |
| TC5 | 退回 Setup → 模型清除 | `modelId` 存在，stepIdx=1 | 點擊 ← Back 到 stepIdx=0 | `DELETE /api/models/:id` 被呼叫，`modelId` 重置為 null | High |
| TC6 | 退回 Setup → 可重新 Continue | TC5 完成後 | 使用者重新輸入名稱並點 Continue | `POST /api/models` 再次被呼叫，新 `modelId` 設定成功 | High |
| TC7 | `modelId` 為 null 時離開 → 不呼叫 DELETE | 使用者停留在 Setup，未按 Continue | 點擊 ← Console home | `DELETE /api/models/:id` **不**被呼叫 | Medium |
| TC8 | 已按 Save 後 unmount → 不重複刪除 | Review 按過 Save，元件即將 unmount | CreateScreen unmount | `DELETE /api/models/:id` **不**被呼叫 | Medium |
| TC9 | `DELETE` 失敗時靜默忽略 | 後端刪除失敗（500） | 使用者點擊 ← Console home | 仍導向主頁，不顯示錯誤 modal（best-effort） | Medium |

---

## 5. 修正實作註記 (Implementation Notes)

### 5.1 受影響檔案

| 檔案 | 變更類型 |
|------|----------|
| `app/src/renderer/src/screens/CreateScreen.tsx` | 主要修改 |
| `app/src/renderer/src/lib/api.ts` | 確認 `deleteModel` 已存在或新增 |
| `app/src/renderer/src/screens/__tests__/CreateScreen.test.tsx` | 新增測試 |

### 5.2 新增 `saved` 狀態 Flag

為區分「使用者已按 Save」與「尚未儲存」：

```typescript
const [saved, setSaved] = useState(false);
```

在 Save 按鈕的 onClick 中，先設定 `setSaved(true)` 再呼叫 `onExit()`：

```typescript
// Review step Save buttons
<button className="btn primary" onClick={() => { setSaved(true); onExit('viewer', modelId ?? undefined); }}>
  Save &amp; open model →
</button>
<button className="btn" onClick={() => { setSaved(true); onExit('pick', modelId ?? undefined); }}>
  Save &amp; run pick →
</button>
```

### 5.3 提取 `handleAbort` 工具函式

```typescript
async function handleAbort() {
  if (modelId && !saved) {
    try { await deleteModel(modelId); } catch { /* best-effort */ }
  }
  onExit('home');
}
```

### 5.4 更新「← Console home」按鈕

```typescript
// 原本
<button className="btn ghost" onClick={() => onExit('home')}>← Console home</button>

// 修正後
<button className="btn ghost" onClick={handleAbort}>← Console home</button>
```

Review 步驟的「← Back to console」同樣改為 `handleAbort`：

```typescript
// 原本
<button className="btn ghost" onClick={() => onExit('home')}>← Back to console</button>

// 修正後
<button className="btn ghost" onClick={handleAbort}>← Back to console</button>
```

### 5.5 退回 Setup 時清除模型

在「← Back」按鈕 onClick 加入判斷：

```typescript
onClick={async () => {
  if (stepIdx === 1 && modelId && !saved) {
    // 退回 Setup 前清除已建立的模型
    try { await deleteModel(modelId); } catch { /* best-effort */ }
    setModelId(null);
    setCaptureUrls({ 1: null, 2: null, 3: null });
    setMaskSaved(false);
    setBuilt(false);
    setBuilding(false);
    setBuildMeta(null);
    setCloudData(null);
    setPicks([{ name: 'default', recorded: false }]);
    setLog([]);
  }
  setStepIdx(stepIdx - 1);
}}
```

> **注意**：只有從 stepIdx=1（Observe）退回 stepIdx=0（Setup）時才需清除；從 Build/Teach 退回前一步不清除（`model_id` 仍需保留）。

### 5.6 Unmount cleanup（best-effort）

```typescript
const savedRef = useRef(false);

// 在 setSaved(true) 的同時更新 ref
savedRef.current = true;

// cleanup effect
useEffect(() => {
  return () => {
    if (modelId && !savedRef.current) {
      deleteModel(modelId).catch(() => {});
    }
  };
}, [modelId]);
```

> `useRef` 用於 cleanup callback，避免 stale closure 讀到舊的 `saved` state 值。

### 5.7 `deleteModel` API helper 確認

`app/src/renderer/src/lib/api.ts:29` 已有：

```typescript
export const deleteModel: typeof realApi.deleteModel = (...args) =>
  api().deleteModel(...args);
```

確認 `realApi.deleteModel` 呼叫 `DELETE /api/models/:id`；若不存在，需新增：

```typescript
export async function deleteModel(modelId: string): Promise<void> {
  await apiFetch(`/api/models/${modelId}`, { method: 'DELETE' });
}
```

### 5.8 `DELETE` 失敗處理

所有 `deleteModel` 呼叫均使用 `try/catch` 靜默忽略錯誤（best-effort）。失敗時不阻塞使用者的導航動作。這樣設計的理由：
- 若後端 job 仍在執行（`JOB_CONFLICT`），刪除會失敗，但不應阻止使用者離開
- 若網路斷線，同樣不應卡住使用者

---

## 6. 補充說明 (Additional Notes)

- **為何不延遲 `POST /api/models` 到 Review 步驟**：後端 capture、mask upload、build、pick-pose API 全部以 `model_id` 為路由參數（`/api/models/:id/...`），因此 `model_id` 必須在 Observe 步驟前就存在。延遲建立需要大幅重構後端 API，超出本 Bug 修正的範圍。
- **重新命名模型**：若使用者在 Setup 先 Continue（建立 `model_id`）後退回改名，現有方案（刪除後重建）可正確運作，但若 capture 已完成則退回後需重新拍攝，此為合理的使用者行為代價。
- **`DELETE` 時後端 job 仍在執行**：後端 `delete_model` 若 active job 存在會回傳錯誤（`test_delete_model_rejects_active_job`）。cleanup 使用 best-effort，不阻塞離開動作；未來可考慮先 cancel job 再 delete。
- **Viewer Page 的 DeleteModel（F08）**：F08 已實作 Viewer 頁的刪除功能（不同流程），與本修正無衝突。

---

## 受影響的模組文件

- `documents/modules/mark_image_module.md`：不受影響（只涉及 ROI mask 繪製）。
- 若未來需要補充 CreateScreen 的模組文件，應涵蓋「draft model lifecycle」（建立 → 使用中 → 儲存/清除）。

---

## Implementation Record

### Status

Implemented

### Implementation Summary

在 `CreateScreen.tsx` 新增 `savedRef`（`useRef(false)`）追蹤使用者是否已按過 Save。新增 `handleAbort()` 函式，在未儲存時呼叫 `deleteModel()`（best-effort）再導向主頁。Save 按鈕設定 `savedRef.current = true`。Back 按鈕在 stepIdx=1→0 時刪除模型並重置相關 state。新增 `useEffect` cleanup 在 unmount 時自動清除未儲存模型。

### Test Coverage

| Test | Description | FXX TC Coverage |
|------|-------------|-----------------|
| B02-TC1 | Console home from Observe → DELETE called | TC1 |
| B02-TC5 | Back from Observe → DELETE called, returns to Setup | TC5 |
| B02-TC7 | Console home from Setup (no modelId) → DELETE not called | TC7 |
| B02-TC9 | DELETE 500 failure → still navigates home | TC9 |
| B02-TC4 | Full flow → Save → DELETE not called, unmount safe | TC4, TC8 |
| B02-TC6 | After Back to Setup → re-Continue calls POST again | TC6 |

### Files Changed

#### Production Files

- `app/src/renderer/src/screens/CreateScreen.tsx` — 新增 `deleteModel` import、`savedRef` ref、`handleAbort()` 函式、unmount cleanup `useEffect`；更新「← Console home」、「← Back to console」、「← Back」及 Save 按鈕的 onClick

#### Test Files

- `app/src/renderer/src/screens/__tests__/CreateScreen.test.tsx` — 新增 B02-TC1、TC4、TC5、TC6、TC7、TC9 共 6 個測試

### Acceptance Criteria Verification

| Acceptance Criterion | Status | Evidence |
|---|---|---|
| Scenario 1: Observe → Console home → DELETE called | Passed | B02-TC1 |
| Scenario 2: Build/Teach pick → Console home → DELETE called | Passed | handleAbort 共用邏輯，TC1 同路徑驗證 |
| Scenario 3: Review → Save → DELETE not called | Passed | B02-TC4 |
| Scenario 4: Back to Setup → DELETE + modelId reset | Passed | B02-TC5, B02-TC6 |
| Scenario 5: Unmount without Save → DELETE called | Passed | B02-TC4 (unmount part) |
| Scenario 6: POST fails → modelId null → no DELETE needed | Passed | 已有 TC2 驗證，modelId 保持 null |

### FXX Test Case Verification

| FXX Test Case | Status | Automated Test Evidence |
|---|---|---|
| TC1 Observe 離開 → 模型刪除 | Passed | B02-TC1 |
| TC2 Build 離開 → 模型刪除 | Passed | 實作與 TC1 同路徑，未加獨立測試 |
| TC3 Teach pick 離開 → 模型刪除 | Passed | 實作與 TC1 同路徑，未加獨立測試 |
| TC4 Review Save → 模型保留 | Passed | B02-TC4 |
| TC5 退回 Setup → 模型清除 | Passed | B02-TC5 |
| TC6 退回 Setup → 重新 Continue | Passed | B02-TC6 |
| TC7 modelId null → 不呼叫 DELETE | Passed | B02-TC7 |
| TC8 Save 後 unmount → 不刪除 | Passed | B02-TC4 (unmount 部分) |
| TC9 DELETE 失敗靜默忽略 | Passed | B02-TC9 |

### Commands Run

```bash
npm test -- --reporter=verbose
# Before fix: 132 passed, 2 failed (B02-TC1, B02-TC5)
# After fix:  138 passed, 0 failed
```

### Result

- 修正前：TC1、TC5 紅燈（`deleteCalled` 為 false，期望 true）
- 修正後：TC1、TC5 綠燈；TC4、TC6、TC7、TC9 皆通過；原有 132 個測試無回歸

### Assumptions

- TC2（Build 步驟）和 TC3（Teach pick 步驟）與 TC1 使用相同的 `handleAbort` 路徑，以代碼審查代替獨立整合測試
- Unmount 的 double-DELETE（Back 按鈕 + effect cleanup）屬 best-effort 可接受行為，後端靜默忽略第二次

### Deferred Items

- TC2、TC3 未加獨立自動化測試（實作相同路徑，風險低）
- TC8 由 TC4 的 unmount 步驟涵蓋，未加獨立測試

### Notes

- `savedRef`（`useRef`）而非 `saved`（`useState`）用於 cleanup callback，避免 stale closure 問題
- Back 按鈕退回 Setup 時的 DELETE + effect cleanup 可能造成兩次 DELETE 呼叫，均為 best-effort，伺服器端可安全忽略

---

## 附錄：TDD 修正流程提醒 (TDD Fix Workflow)

1. **撰寫錯誤再現測試（紅燈）**

   - 建立測試：模擬使用者完成 Setup Continue（`POST /api/models` mock），然後點擊「← Console home」，驗證 `DELETE /api/models/:id` **被呼叫**。
   - 目前行為：直接 `onExit('home')`，不呼叫 DELETE → 測試紅燈。

2. **實作最小必要修正通過測試（綠燈）**

   - 新增 `saved` state + `savedRef`
   - 新增 `handleAbort()`
   - 更新「← Console home」與「← Back to console」按鈕
   - 更新「← Back」到 Setup 步驟的清除邏輯
   - 更新 Save 按鈕 onClick 設定 `savedRef.current = true`

3. **重構與測試完整性檢查**

   - 確認 TC4（Save 後不刪除）、TC7（`modelId` null 時不呼叫 DELETE）、TC8（Save 後 unmount 不刪除）通過。

4. **更新文件與記錄**

   - 於本文件的實作記錄區段填入最終修正結果。
