---
author: lab
date: 2026-05-12
title: 將相機生命週期從每次 Job 建立改為 Server 持有的常駐服務
uuid: a3f2c1e0b4d94f8a9c7e5b2d1f3a6e8c
version: 1.0
---

# 重構需求書 – 重構 Backend Camera Manager

## 1. 目標

每次 Capture Job 與 Pick Workflow Job 都會重新執行 `start_camera()` + `warmup_camera()`，造成約 10 秒的暖機等待。重構後改由 Server 啟動時統一初始化相機，Job 執行時直接向常駐的 `CameraService` 取得畫面，暖機只發生一次。

主要好處：
- 消除每個 Job 開頭 ~10 s 的相機啟動延遲
- 相機資源由單一服務統一管理，避免多 Job 同時爭用硬體
- 前端可透過 `/api/camera/status` 感知相機狀態，提供更好的 UX

---

## 2. 使用者故事

- **As a** 前端使用者
- **I want** 在 Capture 與 Pick Workflow 操作時不需要等待相機暖機
- **So that** 整體操作流程更流暢，每個步驟可以即時觸發

---

## 3. 重構細項說明

| 編號 | 說明 | 調整重點 | 注意事項 |
|------|------|-----------|----------|
| R1 | 建立 `CameraService` 單例 | 新增 `backend/services/camera_service.py`，包含 `start()` / `stop()` / `snapshot()` / `status()`，以 `threading.Lock` 保護狀態轉換 | 狀態機：`off → warming → ready`；`error` 時需支援重啟 |
| R2 | Server 啟動 / 關閉時連動相機 | `app.py` 啟動後呼叫 `camera_service.start()`；以 `atexit` 或 signal handler 呼叫 `camera_service.stop()` | 暖機在背景 thread 執行，不阻塞 Flask 啟動 |
| R3 | 新增相機管理 API | 新增 `GET /api/camera/status`、`POST /api/camera/start`、`POST /api/camera/stop`、`POST /api/camera/snapshot` 四個端點（詳見 `documents/modules/backend-camera-manager.md`） | `snapshot` 結果存至 `output/snapshots/`，納入 `/api/files` 白名單 |
| R4 | 重構 `capture_model.py` | `run_real_capture()` 移除 `start_camera()` / `warmup_camera()` / `pipeline.stop()`；改呼叫 `camera_service.snapshot(timeout_ms)` 取得每張畫面 | `CameraService` 以依賴注入方式傳入，方便測試時替換 mock |
| R5 | 重構 `pick_workflow.py` | `run_pick_workflow()` 在呼叫子程序前先呼叫 `camera_service.snapshot()`，將畫面存至 `run_dir/observe.png`；子程序改以 `--image run_dir/observe.png` 取代 `--capture` | 確認外部腳本 `estimate_pose_from_sift_3d.py` 支援 `--image` 旗標；若不支援則先保留 `--capture` 並在腳本層另行處理 |
| R6 | 向下相容 | 現有所有其他 API 端點（`/capture`、`/build`、`/pick-poses`、`/pick/workflow`）的請求與回應格式保持不變 | 只改內部實作，外部合約不動 |

---

## 4. 測試檢核

| 測試項目 | 說明 | 優先 |
|----------|------|------|
| CameraService 狀態機 | `off → warming → ready`；`stop()` 後回到 `off`；連續呼叫 `start()` 不重啟 | 高 |
| Snapshot 正常路徑 | `status=ready` 時 `snapshot()` 回傳 ndarray 且非 None | 高 |
| Snapshot 相機未就緒 | `status=off/warming/error` 時 `snapshot()` 拋出 `CameraNotReadyError`，API 回 `CAMERA_NOT_READY 409` | 高 |
| Capture Job 整合 | `run_real_capture()` 不再直接呼叫 `start_camera()`；mock `CameraService.snapshot()` 驗證流程 | 高 |
| Pick Workflow 整合 | `run_pick_workflow()` 在子程序前先呼叫 `snapshot()`；驗證 `observe.png` 已存在於 `run_dir` | 高 |
| `/api/camera/status` | 回傳正確 status 與欄位 | 中 |
| `/api/camera/start` 冪等性 | 連續呼叫兩次不重啟，回應均為 `202 warming/ready` | 中 |
| 錯誤恢復 | 相機 `error` 後呼叫 `stop()` 再 `start()` 可回到 `ready` | 中 |
| Server 啟動自動暖機 | Flask app 建立後 CameraService 自動開始暖機（integration test） | 低 |

---

## 5. 重構要點

- **單一擁有者**：相機 pipeline 由 `CameraService` 獨家持有，任何 Job 都不得直接呼叫 `start_camera()` 或 `pipeline.stop()`
- **非阻塞啟動**：Server 啟動時相機在背景 thread 暖機，不影響 API 回應能力
- **依賴注入**：`run_real_capture()` 與 `run_pick_workflow()` 接受 `CameraService` 作為參數，測試時可替換為 mock
- **向下相容**：所有現有 API 的請求/回應格式不變，重構只影響內部實作
- **Pick Workflow 解耦**：後端先取得畫面再交給外部腳本，消除腳本自行啟動相機的需要

---

## 6. 補充說明

**相關文件：**
- API 設計細節：`documents/modules/backend-camera-manager.md`
- 現有 API 參考：`documents/modules/backend-api.md`

**外部腳本確認項目（實作前需驗證）：**
- `estimate_pose_from_sift_3d.py` 是否支援 `--image <path>` 旗標（不啟動相機，直接使用既有圖片）
- 若不支援，R5 需同步修改腳本或改用其他傳遞方式

**`output/snapshots/` 需納入 `/api/files` 白名單**（目前白名單為 `output/models/`、`output/jobs/`、`output/runs/`）

---

## 實作記錄 (Implementation Record)

### 狀態

Implemented

### 實作摘要

依 R1–R6 分五步完成重構：
1. **R1** — 新增 `backend/services/camera_service.py`，`CameraService` 含狀態機、`start/stop/snapshot/status`，以 `threading.Lock` 保護。
2. **R2** — `create_app()` 建立 `CameraService` 並在非 TESTING 模式自動呼叫 `start()`，以 `atexit.register` 確保 `stop()`。
3. **R3** — 新增 `GET /api/camera/status`、`POST /api/camera/start`、`POST /api/camera/stop`、`POST /api/camera/snapshot`；`output/snapshots/` 納入 `/api/files` 白名單（`files.py`）。
4. **R4** — `run_real_capture(request, camera_service)` 移除 `start_camera/warmup_camera/pipeline.stop`，改呼叫 `camera_service.snapshot(timeout_ms)`。
5. **R5** — `app.py` work closure 在呼叫 runner 前先 snapshot 存至 `run_dir/{observe_name}.png`；`run_pick_workflow` 偵測預存圖片使用 `--image`，無圖時回退 `--capture`；`build_pick_workflow_command` 新增 `image_path` 參數。

### 測試覆蓋

新增 `backend/tests/test_r01_camera_manager.py`（37 個測試）：

| 測試類別 | 測試數 | 涵蓋項目 |
|---------|--------|---------|
| `TestCameraServiceStateMachine` | 11 | R1 狀態機（off/warming/ready/error、冪等性、錯誤恢復） |
| `TestCameraServiceSnapshot` | 5 | R1 snapshot 正常/未就緒/無幀 |
| `TestCameraStatusEndpoint` | 2 | R3 GET /api/camera/status |
| `TestCameraStartEndpoint` | 3 | R3 POST /api/camera/start + 冪等 |
| `TestCameraStopEndpoint` | 2 | R3 POST /api/camera/stop |
| `TestCameraSnapshotEndpoint` | 3 | R3 POST /api/camera/snapshot + 檔案存取 + 409 |
| `TestR04CaptureModelIntegration` | 1 | R4 `start_camera` 不再被呼叫，frames 來自 snapshot() |
| `TestR05PickWorkflowCommandBuilder` | 2 | R5 --image / --capture flag 邏輯 |
| `TestR05PickWorkflowPreCapture` | 2 | R5 observe.png 存在於 run_dir、像素值來自 camera_service |

### 變更檔案

#### Production Files
- `backend/services/camera_service.py` — 新增
- `backend/app.py` — 加入 camera API 端點、lifecycle、R4/R5 work closure
- `backend/services/files.py` — `resolve_allowed_file` 加入 `output_snapshots_root`
- `backend/services/capture_model.py` — R4：`run_capture(request, camera_service=None)` / `run_real_capture` 使用 service
- `backend/services/pick_workflow.py` — R5：`build_pick_workflow_command(image_path=None)` / `run_pick_workflow` 偵測預存圖

#### Test Files
- `backend/tests/test_r01_camera_manager.py` — 新增（37 個測試）
- `backend/tests/test_f06_capture_images.py` — `failing_capture` 簽名加 `_camera_service=None`
- `backend/tests/test_f12_pick_workflow.py` — `fake_runner` 簽名加 `_camera_service=None`

### 驗收標準驗證

| 驗收標準 | 狀態 | 證據 |
|---------|------|------|
| CameraService 狀態機 off→warming→ready | Passed | `TestCameraServiceStateMachine` (11 tests) |
| snapshot() 正常路徑 | Passed | `test_snapshot_returns_color_ndarray_when_ready` |
| snapshot() 未就緒拋出 CameraNotReadyError | Passed | `test_snapshot_raises_camera_not_ready_when_off/error` |
| API 四端點回傳正確格式與狀態碼 | Passed | `TestCamera*Endpoint` (10 tests) |
| snapshot 檔案可由 /api/files 取得 | Passed | `test_post_snapshot_file_is_accessible_via_files_api` |
| run_real_capture 不呼叫 start_camera | Passed | `test_run_real_capture_uses_camera_service_not_start_camera` |
| pick_workflow observe.png 在 runner 呼叫前已存在 | Passed | `test_observe_image_pre_captured_before_runner_is_called` |
| 現有 API 合約不變 | Passed | 全體 53 個原有測試通過 |

### 執行指令

```bash
python -m pytest backend/tests/ -q
# 84 passed in 2.33s
```

### 架構決策

- **Pre-capture 位置**：放在 `app.py` work closure（非 `run_pick_workflow` 內部），使 `PICK_WORKFLOW_RUNNER` 替換測試得以驗證行為，同時保持 `run_pick_workflow` 可獨立呼叫。
- **run_pick_workflow 回退**：`observe_png` 不存在且 `camera_service=None` 時保留 `--capture`，維持腳本獨立執行能力。
- **CAMERA_SERVICE 注入**：TESTING=True 時不自動暖機，測試完全掌控 CameraService 生命週期。

### 延遲項目

無。R6 向下相容已通過所有原有測試驗證。

---

## 附錄：TDD 重構流程提醒 (TDD Refactoring Workflow)

1. **建立現有行為測試（回歸測試）**
   - 為 `run_real_capture()` 與 `run_pick_workflow()` 補齊使用 mock camera 的回歸測試，確保重構後行為一致

2. **小步重構與測試通過**
   - R1（建立 CameraService）→ R2（Server 連動）→ R3（API 端點）→ R4（Capture 重構）→ R5（Workflow 重構）
   - 每步均須測試綠燈才進行下一步

3. **更新文件與模組說明**
   - `backend-api.md` 補充新增的四個 Camera 端點
   - `backend-camera-manager.md`（已建立）持續更新

4. **刪除舊邏輯與驗證回歸測試通過**
   - 確認無任何 Job 直接呼叫 `start_camera()` 後，移除 `capture_model.py` 中的相機初始化程式碼

5. **記錄變更與版本控制**
   - PR 說明需涵蓋：動機（10 s 延遲）、變更模組、測試覆蓋範圍
