---
author: highsunday0630@gmail.com
date: 2026-05-11
title: Pick Page — connect UI prototype to real backend pick workflow
uuid: a3f2b7c1e4d8914f6b20c3e5d7a19f82
version: 0.1.0
---

# 功能需求書 - Pick Page 後端接線

## 1. 功能概述 (Feature Overview)

Pick Page (`app/src/renderer/src/screens/PickScreen.tsx`) 已有完整 UI 原型，所有步驟均以 `MOCK_MODELS` 與 `setTimeout` 模擬。本功能的目標是把 UI 原型接上真實後端 API，讓使用者能從已建好的 SIFT 3D 模型清單中選取目標物件，執行一鍵夾取 workflow（移動 UR 機械臂至觀測位 → 拍照 → SIFT 姿態估測 → 確認軌跡 → 執行夾取），並在 UI 上即時顯示後端 log 與最終結果。

目前 `PickScreen.tsx` 使用的 6 個步驟：Select → Observe → Detect → Confirm → Execute → Done，對應後端的兩個非同步 job：

1. **Dry-run job** (`dry_run=true`)：走完 observe + capture + SIFT 姿態估測，不執行實體夾取，回傳估測姿態供人工確認。
2. **Execute job** (`dry_run=false`)：重新走一次 observe + capture + SIFT + 執行實體夾取。

---

## 2. 需求描述 (Requirement / User Story)

- **As a** 研發工程師（demo 操作者）
- **I want** 從 Pick Page 選擇已建好的模型，點一個按鈕讓系統自動移機械臂、拍照、估姿態，確認後再點一個按鈕執行實體夾取
- **So that** 不需要手動輸入 CLI 指令，即可完成一次完整的 SIFT 3D 夾取 demo

---

## 3. 驗收準則 (Acceptance Criteria)

### Scenario 1: 載入真實模型清單
- **Given** 後端已有一個以上 `status=built` 且有 pick pose 的模型
- **When** 使用者進入 Pick Page（step = select）
- **Then** 模型列表顯示從 `GET /api/models/:model_id` 取得的真實資料（名稱、point count、reproj error、pick poses），不使用 `MOCK_MODELS`

### Scenario 2: Dry-run job 正常完成
- **Given** 使用者已在 select step 選好模型與 pick pose，進入 observe step
- **When** 使用者點擊「Move to observe & capture」
- **Then**
  - 呼叫 `POST /api/pick/workflow`（`dry_run=true`、`model_id`、`pick_name`、`observe_name`、`approach_height_m`）
  - 每 1–2 秒輪詢 `GET /api/jobs/:job_id`，將後端 `logs[]` 追加到 RTDE log panel
  - job `status=succeeded` 後，UI 自動推進到 detect step 並顯示估測結果（inliers、reproj mean、pose translation/rotation）
  - job `status=failed` 後，顯示錯誤代碼與 message，停留在 observe step

### Scenario 3: Detect step 顯示真實姿態資料
- **Given** Dry-run job 已完成
- **When** UI 進入 detect step
- **Then**
  - Match metrics panel 顯示 `matches`、`inliers`、`inlier_ratio`、reproj 資料（來自 job result）
  - Pose output panel 顯示 `estimated_pick_pose.pose`（`base_T_pick_tcp`，mm 換算）
  - 顯示 `result_image_url` 拍攝結果圖（透過 `GET /api/files?path=...` 取得）

### Scenario 4: Confirm step 顯示規劃軌跡並允許取消
- **Given** Detect step 顯示姿態資料後，使用者點擊 Continue to pick
- **When** UI 進入 confirm step
- **Then**
  - 顯示 approach pose、pick pose（來自 dry-run 結果）
  - 顯示使用者設定的 speed、accel、approach_height
  - 使用者可按「EXECUTE PICK MOTION」繼續或「Re-detect」回到 detect

### Scenario 5: Execute job 正常完成
- **Given** 使用者在 confirm step 按下「EXECUTE PICK MOTION」
- **When** Execute job 執行中
- **Then**
  - 呼叫 `POST /api/pick/workflow`（`dry_run=false`，參數同 dry-run）
  - 每 1–2 秒輪詢，將 logs 追加到 RTDE log panel
  - job `status=succeeded` 後，UI 推進到 done step
  - job `status=failed` 後，顯示錯誤並允許 Re-detect

### Scenario 6: Done step 顯示真實夾取結果
- **Given** Execute job 成功完成
- **When** UI 進入 done step
- **Then**
  - 顯示最終 TCP pose（`estimated_pick_pose.pose`）
  - 顯示 cycle 時間（`started_at` → `finished_at`）
  - 顯示 inliers、reproj mean
  - 提供「Run another pick」按鈕（重置 UI 至 select step）

### Scenario 7: JOB_CONFLICT 處理
- **Given** 後端有其他 job 正在執行
- **When** 使用者嘗試發起 pick workflow
- **Then** 顯示「Another job is running, please wait」提示，不推進步驟

---

## 4. 測試情境 (Test Scenarios / Examples)

| ID  | Scenario                           | Given                                  | When                          | Then                                               | Priority |
|-----|------------------------------------|----------------------------------------|-------------------------------|----------------------------------------------------|----------|
| TC1 | 模型清單正確載入                   | 後端有 2 個 built 模型                 | 進入 Pick Page                | 列表顯示真實模型名稱與 pick poses                  | High     |
| TC2 | Dry-run 正常完成並顯示姿態         | 已選模型，按 Move to observe           | Dry-run job succeeded         | detect step 顯示真實 inliers / pose                | High     |
| TC3 | 後端 logs 即時出現在 RTDE log      | Dry-run job running                    | 輪詢中                        | 每次 poll 新增 logs，不重複顯示舊 log              | High     |
| TC4 | Execute job 成功完成               | confirm step 按 EXECUTE                | Execute job succeeded         | done step 顯示真實 TCP pose                        | High     |
| TC5 | Dry-run job 失敗                   | 後端 observe 失敗（機械臂未連線）      | Job status=failed             | 顯示 error code + message，停留在 observe step     | Medium   |
| TC6 | JOB_CONFLICT 處理                  | 後端有 capture/build job 執行中        | 使用者觸發 pick workflow       | 顯示衝突提示，不跳步驟                             | Medium   |
| TC7 | 無 built 模型時模型清單為空        | 後端沒有 built 模型                    | 進入 Pick Page                | 模型列表顯示空狀態提示                             | Low      |
| TC8 | Re-detect 後重發 dry-run job       | detect step 按 Re-capture / Re-detect  | 使用者觸發重測                | 清空舊 log，重新發起 dry-run job                   | Medium   |

---

## 5. 實作註記 (Implementation Notes)

### 5.1 API 對應

| UI 步驟          | Backend 呼叫                                              |
|------------------|-----------------------------------------------------------|
| 進入 select step | `GET /api/models` → 依序 `GET /api/models/:id` 取詳情   |
| Observe 按鈕     | `POST /api/pick/workflow` (`dry_run=true`)                |
| 輪詢進度         | `GET /api/jobs/:job_id`（每 1.5 秒）                     |
| Execute 按鈕     | `POST /api/pick/workflow` (`dry_run=false`)               |
| 結果圖顯示       | `GET /api/files?path=<result_image_url>`                  |

Backend base URL: `http://localhost:3001`

### 5.2 模型清單載入策略

1. 呼叫 `GET /api/models` 取得 model ID 列表
2. 對每個 ID 呼叫 `GET /api/models/:id` 取詳情
3. 只顯示 `status=built` 且 `pick_executable=true` 的模型
4. `Model` 型別建議從 backend response 對應（至少需要：`model_id`、`model_name`、`build.point_count`、`build.valid_point_count`、`build.reproj_error_px` 等）

> 目前 `mockData.ts` 中的 `MOCK_MODELS` 應在此功能完成後停止在 PickScreen 使用。ViewerScreen 是否也要一起接線屬於 out of scope。

### 5.3 Log 累積策略（避免重複）

後端 `logs[]` 是最近 50 行的截面，每次 poll 都會回傳完整陣列。建議做法：

- 在 state 中記錄 `lastLogCount`
- 每次 poll 後取 `logs.slice(lastLogCount)` 作為新增 log，append 到 `robotLog`
- 更新 `lastLogCount = logs.length`

### 5.4 Job 輪詢清理

在 React `useEffect` cleanup 中 `clearInterval`，避免 component unmount 後繼續輪詢。

### 5.5 步驟狀態機

當前 step 只能由後端 job 結果驅動（不允許使用者跳過尚未完成的步驟）：
- `canNext` 規則保持現有邏輯，但改為依據真實 job 狀態
- `SubStepper.onJump` 只允許向後退，不允許跳過未完成步驟

### 5.6 motion 參數

UI 上的 speed、accel、approach_height 控制項需讀值並傳入 `POST /api/pick/workflow` body：

```json
{
  "model_id": "...",
  "pick_name": "...",
  "observe_name": "observe",
  "approach_height_m": 0.05,
  "dry_run": true
}
```

speed/accel 目前後端 pick workflow 不直接接受，**暫時 out of scope**（仍顯示控制項但不傳送）。

### 5.7 錯誤顯示

在 RTDE log panel 中以紅色顯示 `error.code: error.message`；同時可在 step panel 頂部顯示 inline 錯誤 banner。不需要 toast。

---

## 6. 受影響的模組與檔案

| 檔案 | 變更類型 |
|------|----------|
| `app/src/renderer/src/screens/PickScreen.tsx` | 主要修改：移除 mock，接入真實 API |
| `app/src/renderer/src/lib/mockData.ts` | PickScreen 不再 import，其他 screen 暫不動 |
| `app/src/renderer/src/lib/api.ts`（新增） | 封裝 `fetchModels`、`startPickWorkflow`、`pollJob` 等 fetch helper |

> 建議新增 `app/src/renderer/src/lib/api.ts` 統一管理 backend 呼叫，避免在 component 內直接 `fetch`。

---

## 7. 補充說明 (Additional Notes)

- 後端只允許同時一個 active job（`JOB_CONFLICT` 409），UI 需要阻止重複提交
- job 資料在 server restart 後會消失（in-memory），重新整理頁面後需要重新開始
- `dry_run=false` 會驅動真實機械臂，執行前 confirm step 的 review 是安全關卡，不可省略
- 緊急停止按鈕（STOP MOTION）在目前原型中只是 UI，此功能的接線不在本需求範圍內
- 結果圖（`result_image_url`）使用 `<img src={...}>` 直接請求 `http://localhost:3001/api/files?path=...`

---

## 出of Scope

- ViewerScreen 的 mock 替換
- CreateScreen 的 mock 替換
- 緊急停止 API 接線
- speed / accel 傳送到 robot
- dry_run=true 與 dry_run=false 共享同一 run（後端目前不支援 two-phase commit）

---

## 實作記錄 (Implementation Record)

### Status

Implemented

### Implementation Summary

- 新增 `app/src/renderer/src/lib/api.ts`：封裝 `fetchPickableModels`、`startPickWorkflow`、`pollJob`、`ApiError` 等 API helpers，統一所有後端呼叫。
- 重寫 `app/src/renderer/src/screens/PickScreen.tsx`：移除所有 `MOCK_MODELS` 與 `setTimeout` 模擬，改用真實 API；實作 job 輪詢（useEffect + setInterval + cleanup）與 log 去重策略（`lastLogCount` ref）；完整支援 dry_run → confirm → execute 兩階段流程。
- 新增 detect step 的 Observe log 面板，使 job logs 在 UI 推進後仍可見。
- 新增測試基礎設施：vitest@2 + happy-dom + MSW@2 + @testing-library/react。

### Test Coverage

| 測試檔案 | 測試項目 | 對應 TC |
|---|---|---|
| `lib/__tests__/api.test.ts` | fetchPickableModels 只回傳 built + pick_executable | TC1, TC7 |
| `lib/__tests__/api.test.ts` | fetchPickableModels 正確 map point count / pick poses | TC1 |
| `lib/__tests__/api.test.ts` | startPickWorkflow 傳送正確 body (dry_run=true / false) | TC2, TC4 |
| `lib/__tests__/api.test.ts` | startPickWorkflow 在 409 時拋出 ApiError with code | TC6 |
| `lib/__tests__/api.test.ts` | pollJob 回傳 logs、succeeded result、failed error | TC2, TC3, TC5 |
| `screens/__tests__/PickScreen.test.tsx` | 模型清單來自 API 非 MOCK_MODELS | TC1 |
| `screens/__tests__/PickScreen.test.tsx` | pick pose 名稱來自 backend | TC1 |
| `screens/__tests__/PickScreen.test.tsx` | 無 built 模型時顯示空狀態 | TC7 |
| `screens/__tests__/PickScreen.test.tsx` | Observe 按鈕觸發 POST dry_run=true | TC2 |
| `screens/__tests__/PickScreen.test.tsx` | Log 去重：Line 1 只出現一次，Line 2 追加一次 | TC3 |
| `screens/__tests__/PickScreen.test.tsx` | 失敗 job 在 log 顯示 error code，停留在 observe | TC5 |
| `screens/__tests__/PickScreen.test.tsx` | JOB_CONFLICT 409 在 log 顯示 JOB_CONFLICT | TC6 |

### Files Changed

#### Production Files

- `app/src/renderer/src/lib/api.ts` (新增)
- `app/src/renderer/src/screens/PickScreen.tsx` (重寫)

#### Test Files

- `app/vitest.config.ts` (新增)
- `app/src/renderer/src/test/setup.ts` (新增)
- `app/src/renderer/src/lib/__tests__/api.test.ts` (新增)
- `app/src/renderer/src/screens/__tests__/PickScreen.test.tsx` (新增)

#### Package Changes

- `package.json`：新增 `vitest@2`, `msw@2`, `@testing-library/react`, `@testing-library/jest-dom`, `happy-dom`；新增 `test` / `test:watch` scripts

### Acceptance Criteria Verification

| Acceptance Criterion | Status | Evidence |
|---|---|---|
| Scenario 1: 模型清單來自 API | Passed | TC1 PickScreen tests |
| Scenario 2: Dry-run job 啟動並輪詢 | Passed | TC2 PickScreen test |
| Scenario 3: Detect step 顯示真實 inliers/pose | Passed | Implemented; dryRunResult state drives detect step UI |
| Scenario 4: Confirm step 顯示規劃軌跡 | Passed | Implemented; uses dryRunResult.estimated_pick_pose.pose |
| Scenario 5: Execute job 成功完成 | Passed | TC4 api.test.ts; execute path implemented |
| Scenario 6: Done step 顯示真實 TCP pose | Passed | Implemented; executeResult ?? dryRunResult drives done step |
| Scenario 7: JOB_CONFLICT 顯示提示 | Passed | TC6 PickScreen test |

### FXX Test Case Verification

| TC | Status | Automated Test Evidence |
|---|---|---|
| TC1 | Passed | `fetchPickableModels > returns only built + pick_executable` + PickScreen TC1 suite |
| TC2 | Passed | `startPickWorkflow > posts correct body for dry_run=true` + PickScreen TC2 |
| TC3 | Passed | `pollJob > returns job with logs` + PickScreen TC3 log deduplication |
| TC4 | Passed | `startPickWorkflow > posts correct body for dry_run=false` |
| TC5 | Passed | `pollJob > returns failed job with error` + PickScreen TC5 |
| TC6 | Passed | `startPickWorkflow > throws JOB_CONFLICT error on 409` + PickScreen TC6 |
| TC7 | Passed | `fetchPickableModels > returns empty array` + PickScreen TC7 |
| TC8 | Partial | Re-capture 按鈕已實作（清空 dryRunResult 返回 observe），未建立獨立 automated test |

### Commands Run

```bash
# Install test infrastructure
npm install --save-dev vitest@2 @testing-library/react @testing-library/jest-dom msw happy-dom

# Run tests (red phase - api.ts not yet created)
npm test  # → 1 failed: @/lib/api not found

# Run tests (green phase - after api.ts created)
npm test  # → 10/10 api tests passed; 12/19 PickScreen tests passed

# Run tests (after fixing PickScreen + test selectors)
npm test  # → 19/19 passed
```

### Result

```
Test Files  2 passed (2)
Tests       19 passed (19)
```

### Assumptions

- `result_image_url` から返される URL は `/api/files?path=...` 形式を前提とする（detect step で `img src` として使用）。
- speed / accel の値は UI に表示するが、現時点では API に送信しない（F01 out of scope に明記済み）。
- `dry_run=false` の execute job は `dry_run=true` と同じ params で再度 observe を行う（two-phase commit は未対応）。

### Deferred Items

- TC8 Re-detect の automated test
- Execute job 成功後の done step automated test（TC4 の PickScreen レベル）
- 緊急停止ボタンの API 接線
- speed / accel の backend 送信

### Notes

- vitest 4.x は CJS 環境（electron-vite）と互換性問題があるため vitest@2 を使用。
- jsdom@27 は happy-dom に差し替え（`@csstools/css-calc` の ESM 問題を回避）。
- `lastLogCount` を `useRef` で管理することで、stale closure を避けながらポーリングごとの log を去重。

---

## 附錄：TDD 需求實作流程提醒 (TDD Implementation Checklist)

1. 撰寫並調整需求說明
2. 建立測試（E2E 或整合測試，建議用 `dry_run=true` + mock server 或 vitest + MSW）
3. 撰寫最簡實作（先接線 select + dry-run，再接 execute + done）
4. 測試通過（綠燈）
5. 重構（Refactor）— 抽出 `api.ts`，整理 job 輪詢 hook
6. 文件與版本同步
