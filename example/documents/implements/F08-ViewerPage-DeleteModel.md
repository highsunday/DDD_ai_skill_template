---
author: highsunday0630@gmail.com
date: 2026-05-13
title: Viewer Page — delete model from model library
uuid: 8a0c364bd2244f808400aadab3a50d8e
version: 0.1.0
---

# 功能需求書 - Viewer Page 刪除模型

## 1. 功能概述 (Feature Overview)

Viewer Page (`app/src/renderer/src/screens/ViewerScreen.tsx`) 左側 Model library 目前只能選擇已建立完成的模型，用來檢視點雲、拍照影像、pick poses 與 raw metadata。若使用者建立了錯誤、不需要、或測試用模型，目前沒有 GUI 操作可以清理 `output/models/<model_id>` 內的模型檔案。

本功能目標：在 Viewer Page 左側模型清單新增「Delete model」操作。使用者選擇模型後可按刪除按鈕，前端必須先顯示不可復原的 warning popout；只有使用者在 warning 中再次確認 Delete 後，才呼叫後端 API，由後端完整刪除該模型資料夾與其檔案。

刪除範圍包含該模型目錄下所有資料，例如：

- `model.json`
- `sift_3d_model.json`
- `triangulated_points.json`
- `point_cloud_view.png`
- `build_log.txt`
- `captures/`
- `masks/`
- `.build_tmp/`
- 其他位於 `output/models/<model_id>/` 內的檔案或子目錄

## 2. 需求描述 (Requirement / User Story)

- **As a** 研發工程師或 demo 操作者
- **I want** 在 Viewer Page 的 Model library 中刪除選定模型
- **So that** 可以從系統中永久清除不需要的 model files，避免模型清單混亂與磁碟空間浪費

## 3. 驗收準則 (Acceptance Criteria)

### Scenario 1: 選定模型後顯示 Delete model 按鈕

- **Given** Viewer Page 已載入一個以上 built model
- **When** 使用者在左側 Model library 選擇某個模型
- **Then**
  - 左側模型清單區域顯示 `Delete model` 按鈕
  - 按鈕狀態對應目前選定的模型
  - 未選擇模型、模型清單載入中、或刪除進行中時，按鈕不可執行

### Scenario 2: 第一次按 Delete model 只顯示 warning popout

- **Given** 使用者已選擇模型 `my_part`
- **When** 使用者按下 `Delete model`
- **Then**
  - 前端顯示 warning popout / confirmation dialog
  - dialog 明確顯示將刪除的 `model_name` 與 `model_id`
  - dialog 明確告知此操作會永久刪除模型檔案且不可復原
  - 此時不得呼叫後端刪除 API

### Scenario 3: 使用者取消 warning 時不刪除任何檔案

- **Given** warning dialog 已開啟
- **When** 使用者按 `Cancel`、關閉 dialog、或按 Escape
- **Then**
  - dialog 關閉
  - 選定模型仍留在 Viewer Page
  - 不呼叫 `DELETE /api/models/:model_id`
  - backend filesystem 不變

### Scenario 4: 使用者確認 Delete 後後端永久刪除模型

- **Given** warning dialog 已開啟，且模型 `my_part` 存在於 `output/models/my_part`
- **When** 使用者在 dialog 中按確認刪除
- **Then**
  - 前端呼叫 `DELETE /api/models/my_part`
  - 後端驗證 `model_id` 合法且模型存在
  - 後端完整刪除 `output/models/my_part/` 目錄與其所有內容
  - API 回傳成功後，前端從模型清單移除 `my_part`
  - 若清單仍有其他模型，前端自動選擇下一個可用模型；若沒有模型，顯示既有空狀態

### Scenario 5: 後端刪除失敗時保留 UI 狀態

- **Given** 使用者確認刪除模型
- **When** 後端回傳錯誤，例如 `MODEL_NOT_FOUND`、`INVALID_MODEL_ID`、`MODEL_DELETE_FAILED` 或 `JOB_CONFLICT`
- **Then**
  - 前端顯示錯誤訊息
  - 被刪除的模型不得從前端清單中移除，除非後端明確回傳成功
  - 使用者仍可取消 dialog 或重試刪除

### Scenario 6: 有進行中 job 時不得刪除模型

- **Given** backend job store 中存在 queued/running 的 capture、build、save pick pose 或 pick workflow job，且 job 可能使用該模型
- **When** 使用者確認刪除同一模型
- **Then**
  - 後端拒絕刪除並回傳 `JOB_CONFLICT` 與 HTTP 409
  - 模型資料夾不得被刪除
  - 前端顯示「model is busy」類型錯誤

### Scenario 7: Demo Mode 也支援刪除流程

- **Given** `VITE_DEMO_MODE=1`
- **When** 使用者在 Viewer Page 刪除 demo model
- **Then**
  - 前端 warning、確認、清單移除與自動選取行為和 Real Mode 一致
  - mock API 僅從 in-memory demo model store 移除模型
  - 不修改 `demo_mode_data/saved_models/**` fixture 檔案

## 4. 測試情境 (Test Scenarios / Examples)

| ID | Scenario | Given | When | Then | Priority |
|----|----------|-------|------|------|----------|
| TC1 | 顯示刪除按鈕 | Viewer 有 built model 且已選定 | 進入 Viewer Page | 左側顯示可用的 Delete model 按鈕 | High |
| TC2 | 開啟 warning 不呼叫 API | 已選擇 `part_a` | 按 Delete model | dialog 顯示 model name/id，API 尚未被呼叫 | High |
| TC3 | 取消刪除 | warning dialog 已開啟 | 按 Cancel 或 Escape | dialog 關閉，模型仍存在，API 未呼叫 | High |
| TC4 | 確認刪除成功 | backend 有 `output/models/part_a` | 在 warning 中按 Delete | 呼叫 DELETE API，資料夾不存在，清單移除 `part_a` | High |
| TC5 | 刪除後自動選下一個模型 | 清單有 `part_a`、`part_b`，選擇 `part_a` | 成功刪除 `part_a` | UI 選定 `part_b` 並載入其資料 | Medium |
| TC6 | 刪除最後一個模型 | 清單只有 `part_a` | 成功刪除 `part_a` | Viewer 顯示 No models found 空狀態 | Medium |
| TC7 | 後端失敗不移除 UI model | DELETE 回傳 500 或 `MODEL_DELETE_FAILED` | 確認刪除 | 顯示錯誤，`part_a` 仍在清單中 | High |
| TC8 | 模型不存在 | `model.json` 已不存在 | 呼叫 DELETE | 後端回傳 404 `MODEL_NOT_FOUND` | Medium |
| TC9 | 路徑安全 | request 使用 `../bad` 類型 model_id | 呼叫 DELETE | 後端回傳 400 `INVALID_MODEL_ID`，不刪除 output root 外檔案 | High |
| TC10 | 進行中 job 阻擋刪除 | 同一 model 有 queued/running job | 呼叫 DELETE | 後端回傳 409 `JOB_CONFLICT`，資料夾仍存在 | High |
| TC11 | Demo Mode 刪除 | demo model `mycard` 存在 | 確認刪除 | mock 清單移除 `mycard`，fixture files 不變 | Medium |

## 5. 實作註記 (Implementation Notes)

### 5.1 Frontend 行為

主要修改 `app/src/renderer/src/screens/ViewerScreen.tsx`：

- 在左側 `Panel title="Saved models"` 的模型清單下方新增 `Delete model` 按鈕。
- 按鈕應只對目前 `selectedModel` 生效。
- 按下按鈕後以 modal/dialog 呈現 warning，不直接刪除。
- dialog 內容至少包含：
  - `model_name`
  - `model_id`
  - 永久刪除、不可復原、會刪除 backend model files 的警告文字
  - `Cancel`
  - destructive style 的 `Delete`
- 確認刪除時：
  - 設定 deleting/loading 狀態，避免重複送出
  - 呼叫 API facade 的 `deleteModel(modelId)`
  - 成功後更新 `viewerModels`
  - 若刪除的是目前選定模型，選擇清單中的下一個模型；沒有下一個時將 `selected` 設為 `null`
  - 清除該模型的 `cloudData`、`capturesData` 與 error/loading state
  - 失敗時顯示錯誤，不移除清單項目

### 5.2 Frontend API facade

需擴充：

| 檔案 | 新增內容 |
|------|----------|
| `app/src/renderer/src/lib/api.ts` | export `deleteModel` facade |
| `app/src/renderer/src/lib/realApi.ts` | 實作 `deleteModel(modelId: string)`，呼叫 `DELETE /api/models/:model_id` |
| `app/src/renderer/src/lib/mockApi.ts` | 實作 demo mode in-memory delete |
| `app/src/renderer/src/lib/apiTypes.ts` | 如需共用 response type，可新增 `DeleteModelResult` |

建議 API helper：

```typescript
export interface DeleteModelResult {
  model_id: string;
  deleted: true;
}

export async function deleteModel(modelId: string): Promise<DeleteModelResult>
```

### 5.3 Backend API

新增 endpoint：

```http
DELETE /api/models/:model_id
```

成功 response：

```json
{
  "ok": true,
  "data": {
    "model_id": "my_part",
    "deleted": true
  }
}
```

錯誤 response：

| Code | HTTP | Meaning |
|------|------|---------|
| `INVALID_MODEL_ID` | 400 | `model_id` 不合法或包含不安全路徑 |
| `MODEL_NOT_FOUND` | 404 | 模型不存在 |
| `JOB_CONFLICT` | 409 | 有 queued/running job 正在使用該模型或 model operation |
| `MODEL_DELETE_FAILED` | 500 | filesystem delete 失敗 |

### 5.4 Backend deletion service

建議在 `backend/services/models.py` 新增 service function，例如：

```python
def delete_model(output_models_root: Path, model_id: str) -> None:
    root = model_root(output_models_root, model_id)
    metadata_path = root / "model.json"
    if not metadata_path.exists():
        raise BackendError("MODEL_NOT_FOUND", f"Model {model_id} does not exist.", 404)
    shutil.rmtree(root)
```

安全限制：

- 必須沿用 `validate_model_id()` / `model_root()` 的路徑安全檢查。
- 只允許刪除 `OUTPUT_MODELS_ROOT` 內對應 `model_id` 的目錄。
- 不允許刪除任意 request path。
- 不接受 request body 指定刪除路徑。
- `shutil.rmtree` 例外需包成 `BackendError("MODEL_DELETE_FAILED", ..., 500)`。

### 5.5 Job conflict 規則

刪除前後端需檢查是否有 active operation。

最小可接受規則：

- 若 `JobStore.has_active_operation({"build_model", "capture_model", "save_pick_pose", "pick_workflow"})` 為 true，拒絕刪除並回傳 `JOB_CONFLICT`。

較精準的後續改善：

- 若 JobStore 未來支援依 `model_id` 查詢 active job，則只阻擋同一 model 的 active job。

### 5.6 Backend API docs

實作完成後需更新 `documents/modules/backend-api.md` 的 Models section，新增 `DELETE /api/models/:model_id` 文件與錯誤碼。

## 6. 受影響的模組與檔案

| 檔案 | 變更類型 |
|------|----------|
| `app/src/renderer/src/screens/ViewerScreen.tsx` | 新增 Delete model 按鈕、warning dialog、刪除成功/失敗 UI state |
| `app/src/renderer/src/lib/api.ts` | 新增 delete API facade |
| `app/src/renderer/src/lib/realApi.ts` | 新增 Real Mode deleteModel helper |
| `app/src/renderer/src/lib/mockApi.ts` | 新增 Demo Mode in-memory deleteModel helper |
| `app/src/renderer/src/lib/apiTypes.ts` | 視需要新增 DeleteModelResult type |
| `backend/app.py` | 新增 `DELETE /api/models/<model_id>` route |
| `backend/services/models.py` | 新增 delete_model service，負責安全刪除模型資料夾 |
| `backend/tests/test_f02_api.py` 或新測試檔 | 新增 backend delete model API 測試 |
| `app/src/renderer/src/lib/__tests__/api.test.ts` | 新增 real API helper 測試 |
| `app/src/renderer/src/lib/__tests__/mockApi.test.ts` | 新增 demo mode delete 測試 |
| `app/src/renderer/src/screens/__tests__/ViewerScreen.test.tsx` | 新增 Viewer delete button/dialog/UI state 測試 |
| `documents/modules/backend-api.md` | 實作後需更新 API 文件 |
| `documents/modules/GUI設計規劃.md` | 實作後需更新 Viewer Workflow 說明 |

## 7. 補充說明 (Additional Notes)

- 本需求是永久刪除，不提供 recycle bin、archive、undo 或 soft delete。
- 本需求不刪除 `output/jobs/`、`output/runs/`、`output/snapshots/` 中的歷史 job/run/snapshot 檔案。
- 本需求不處理 OS 權限不足或檔案被外部程式鎖定以外的進階復原流程；遇到 filesystem exception 時回傳 `MODEL_DELETE_FAILED`。
- Delete model 操作應放在 Viewer 的 model library 區域，不放在 Pick workflow。
- `+ Import .json` 與 `Export` 仍不在本需求範圍內。

## Out of Scope

- 批次刪除多個模型
- Soft delete / archive / restore
- 刪除 job history、pick run history 或 snapshot history
- 刪除 demo fixture 實體檔案
- 使用者權限與角色控管

## 實作記錄 (Implementation Record)

### Status

Implemented

### Implementation Summary

- Viewer Page 左側 Model library 新增 `Delete model` 按鈕，按下後開啟 warning dialog，顯示 `model_name`、`model_id` 與永久刪除警告。
- 使用者確認後才呼叫 API facade 的 `deleteModel(modelId)`；成功後從清單移除模型，並自動選擇下一個可用模型或顯示既有 empty state。
- 刪除失敗時保留模型清單與 dialog，並顯示後端錯誤訊息。
- Real Mode 新增 `DELETE /api/models/:model_id` backend endpoint，後端以 `model_root()`/`validate_model_id()` 限制刪除範圍，只刪除 `OUTPUT_MODELS_ROOT/<model_id>`。
- Backend delete 會拒絕同一 model 的 active capture/build/save-pick-pose/pick-workflow job，回傳 `JOB_CONFLICT`。
- Demo Mode 新增 in-memory `deleteModel`，只移除 mock model state，不修改 fixture files。
- 已同步更新 `documents/modules/backend-api.md` 與 `documents/modules/GUI設計規劃.md`。

### Test Coverage

| 測試檔案 | 測試項目 | 對應 TC |
|---|---|---|
| `backend/tests/test_f02_api.py` | `DELETE /api/models/:id` recursively removes model workspace and list no longer includes it | TC4 |
| `backend/tests/test_f02_api.py` | missing model returns 404 `MODEL_NOT_FOUND` | TC8 |
| `backend/tests/test_f02_api.py` | unsafe model id is rejected and outside files remain untouched | TC9 |
| `backend/tests/test_f02_api.py` | active model job returns 409 `JOB_CONFLICT` and keeps workspace | TC10 |
| `app/src/renderer/src/lib/__tests__/api.test.ts` | `deleteModel` sends DELETE and returns delete result | TC4 |
| `app/src/renderer/src/lib/__tests__/api.test.ts` | `deleteModel` throws `ApiError` on backend failure | TC7 |
| `app/src/renderer/src/lib/__tests__/mockApi.test.ts` | demo `deleteModel` removes only in-memory built model | TC11 |
| `app/src/renderer/src/screens/__tests__/ViewerScreen.test.tsx` | Delete model button opens warning dialog without calling API | TC1, TC2 |
| `app/src/renderer/src/screens/__tests__/ViewerScreen.test.tsx` | Cancel closes dialog and does not call API | TC3 |
| `app/src/renderer/src/screens/__tests__/ViewerScreen.test.tsx` | Confirm delete removes selected model and selects next model | TC4, TC5 |
| `app/src/renderer/src/screens/__tests__/ViewerScreen.test.tsx` | Delete failure keeps model and shows error | TC7 |
| `app/src/renderer/src/screens/__tests__/ViewerScreen.test.tsx` | Deleting last model shows empty state | TC6 |

### Files Changed

#### Production Files

- `backend/app.py`
- `backend/services/models.py`
- `app/src/renderer/src/lib/api.ts`
- `app/src/renderer/src/lib/apiTypes.ts`
- `app/src/renderer/src/lib/realApi.ts`
- `app/src/renderer/src/lib/mockApi.ts`
- `app/src/renderer/src/screens/ViewerScreen.tsx`

#### Test Files

- `backend/tests/test_f02_api.py`
- `app/src/renderer/src/lib/__tests__/api.test.ts`
- `app/src/renderer/src/lib/__tests__/mockApi.test.ts`
- `app/src/renderer/src/screens/__tests__/ViewerScreen.test.tsx`

#### Documentation Files

- `documents/implements/F08-ViewerPage-DeleteModel.md`
- `documents/modules/backend-api.md`
- `documents/modules/GUI設計規劃.md`

### Acceptance Criteria Verification

| Acceptance Criterion | Status | Evidence |
|---|---|---|
| Scenario 1: 選定模型後顯示 Delete model 按鈕 | Passed | `ViewerScreen — F08: delete selected model > TC1-TC2` |
| Scenario 2: 第一次按 Delete model 只顯示 warning popout | Passed | `ViewerScreen — F08: delete selected model > TC1-TC2` |
| Scenario 3: 使用者取消 warning 時不刪除任何檔案 | Passed | `ViewerScreen — F08: delete selected model > TC3` |
| Scenario 4: 使用者確認 Delete 後後端永久刪除模型 | Passed | `F02ApiTests.test_delete_model_removes_workspace_recursively`, `ViewerScreen — F08 > TC4-TC5` |
| Scenario 5: 後端刪除失敗時保留 UI 狀態 | Passed | `deleteModel > TC7`, `ViewerScreen — F08 > TC7` |
| Scenario 6: 有進行中 job 時不得刪除模型 | Passed | `F02ApiTests.test_delete_model_rejects_active_model_job` |
| Scenario 7: Demo Mode 也支援刪除流程 | Passed | `mockApi demo mode fixtures > F08 TC11` |

### FXX Test Case Verification

| FXX Test Case | Status | Automated Test Evidence |
|---|---|---|
| TC1 | Passed | `ViewerScreen — F08: delete selected model > TC1-TC2` |
| TC2 | Passed | `ViewerScreen — F08: delete selected model > TC1-TC2` |
| TC3 | Passed | `ViewerScreen — F08: delete selected model > TC3` |
| TC4 | Passed | `F02ApiTests.test_delete_model_removes_workspace_recursively`, `deleteModel > TC4`, `ViewerScreen — F08 > TC4-TC5` |
| TC5 | Passed | `ViewerScreen — F08: delete selected model > TC4-TC5` |
| TC6 | Passed | `ViewerScreen — F08: delete selected model > TC6` |
| TC7 | Passed | `deleteModel > TC7`, `ViewerScreen — F08 > TC7` |
| TC8 | Passed | `F02ApiTests.test_delete_model_missing_returns_404` |
| TC9 | Passed | `F02ApiTests.test_delete_model_blocks_unsafe_model_id` |
| TC10 | Passed | `F02ApiTests.test_delete_model_rejects_active_model_job` |
| TC11 | Passed | `mockApi demo mode fixtures > F08 TC11` |

### Commands Run

```bash
pytest backend/tests/test_f02_api.py -q
npm test -- src/renderer/src/lib/__tests__/api.test.ts src/renderer/src/lib/__tests__/mockApi.test.ts src/renderer/src/screens/__tests__/ViewerScreen.test.tsx
pytest backend/tests -q
npm test
```

### Result

- Red phase observed before implementation:
  - `pytest backend/tests/test_f02_api.py -q`: 4 F08 backend tests failed because `DELETE /api/models/:model_id` was not implemented.
  - `npm test -- src/renderer/src/lib/__tests__/api.test.ts src/renderer/src/lib/__tests__/mockApi.test.ts src/renderer/src/screens/__tests__/ViewerScreen.test.tsx`: F08 tests failed because `deleteModel` was not exported and Viewer had no Delete model button/dialog.
- Green phase after implementation:
  - `pytest backend/tests/test_f02_api.py -q`: 17 passed.
  - Targeted frontend command: 64 passed.
  - `pytest backend/tests -q`: 88 passed.
  - `npm test`: 117 passed.

### Assumptions

- Backend blocks deletion only for active jobs on the same `model_id`; `JobStore` already supports model-specific active operation checks.
- Delete removes only `output/models/<model_id>/`; job history, run history, snapshots, and demo fixture files remain out of scope.

### Deferred Items

- No archive, undo, restore, batch delete, or permission model was added.
- Existing `+ Import .json` and `Export` buttons remain unimplemented, as specified out of scope.
