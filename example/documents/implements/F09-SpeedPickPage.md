---
author: highsunday0630@gmail.com
date: 2026-05-14
title: Speed Pick Page - one-button model TCP pick workflow
uuid: 6c9d54207e2c4b1d8f0a37b2c9a86514
version: 0.1.0
---

# 功能需求書 - Speed Pick Page

## 1. 功能概述 (Feature Overview)

新增 **05 Speed Pick** 頁簽，提供比 **04 Pick** 更快的一鍵 pick 流程。

現有 **04 Pick** 是 step-by-step workflow，使用者需要依序選模型、observe/capture、detect review、execute、done。這對 demo review 和安全確認很有用，但在使用者已經熟悉模型與 TCP 後，操作流程太長。

**05 Speed Pick** 目標是讓使用者只做三件事：

1. 在左側選擇模型。
2. 在右側選擇該模型要使用的 pick TCP。
3. 按 **Run**，讓 robot 自動完成 observe、capture、detect、execute pick。

頁面必須顯示 detection result image，並明確顯示 detection 是否 good。執行期間必須鎖住模型/TCP/Run 操作，也必須鎖住頁面切換，避免使用者切到其他頁面再按另一個 Run button。

Layout 必須保持 mechanic simple：左側約 1/4 寬度固定作為 model select rail，右側約 3/4 寬度作為單一 workcell panel，集中顯示 model info、TCP selector、Run、detection result image/status、compact RTDE log。使用者在一般 desktop viewport 不應需要往下捲動才看得到 result。

本功能以 `documents/implements/speed_pick_page_design.md` 為設計來源。若本文件與設計文件有差異，實作時以本 F09 requirement 作為驗收依據。

## 2. 需求描述 (Requirement / User Story)

- **As a** demo 操作者或 production test user
- **I want** 在 05 Speed Pick 頁面選擇已訓練模型與 pick TCP，然後按一次 Run
- **So that** robot 可以自動完成完整 pick workflow，不需要像 04 Pick 一樣逐步操作

## 3. 驗收準則 (Acceptance Criteria)

### Scenario 1: 顯示 05 Speed Pick 頁簽

- **Given** frontend app 已啟動
- **When** 使用者查看 top navigation
- **Then**
  - `05 · Speed Pick` 頁簽顯示在 `04 · Pick` 後面
  - 點擊 `05 · Speed Pick` 會進入 Speed Pick page
  - 不移除、不替換現有 `04 · Pick`

### Scenario 2: 左側載入可 pick 模型

- **Given** backend 有至少一個 `status=built` 且 `pick_executable=true` 的模型
- **When** 使用者進入 Speed Pick page
- **Then**
  - 左側 `Choose model` panel 顯示可 pick 模型清單
  - 模型清單資料來源與 04 Pick 相同，使用 `fetchPickableModels()`
  - 每個模型 row 顯示 model name、point count、valid point count、pick pose count
  - 預設選取第一個可 pick 模型

### Scenario 3: 支援 initial model payload

- **Given** 使用者從其他頁面帶著 model id 進入 Speed Pick
- **When** 該 model id 存在於 pickable model list
- **Then** Speed Pick 預設選取該模型

### Scenario 4: 右側顯示模型資訊與 TCP selector

- **Given** 左側已有選定模型
- **When** 右側 setup panel render
- **Then**
  - 顯示 model name、model id、point count、valid point count、pick pose count
  - 若有 capture image，顯示第一張可用 object image
  - 顯示 `Pick TCP` selector
  - selector options 來自 selected model 的 `pick_poses[].name`
  - 多 TCP 模型可以切換 TCP
  - 單 TCP 模型 selector 為 disabled 或 read-only
  - model info、TCP selector、Run、result/status 位於同一個右側 workcell panel，避免 result 被推到頁面下方

### Scenario 5: 切換模型或 TCP 會清掉舊結果

- **Given** Speed Pick page 已有先前 run 的 result 或 error
- **When** 使用者切換模型或 TCP
- **Then**
  - 清除舊 result
  - 清除舊 error
  - 切換模型時，TCP 重設為該模型第一個 pick pose
  - 不自動啟動 detect 或 pick

### Scenario 6: Run 啟動完整 pick workflow

- **Given** 使用者已選擇 model 與 pick TCP
- **When** 使用者按下 `Run`
- **Then**
  - frontend 呼叫 `POST /api/pick/workflow`
  - request body 包含：

```json
{
  "model_id": "<selected model id>",
  "pick_name": "<selected TCP>",
  "observe_name": "observe",
  "approach_height_m": 0.05,
  "dry_run": false
}
```

- **And** `dry_run=false`，因為 Speed Pick 要執行完整 robot pick workflow
- **And** page 開始 poll `GET /api/jobs/:job_id`

### Scenario 7: Job 執行中鎖定頁面操作

- **Given** Speed Pick workflow job 正在 queued 或 running
- **When** 使用者嘗試操作頁面
- **Then**
  - model list 不可切換
  - TCP selector 不可切換
  - Run button disabled
  - `Open in 04 Pick` 或其他離開 Speed Pick 的 action disabled 或 ignored
  - top navigation、brand/home navigation disabled 或 ignored
  - app 必須維持在 Speed Pick page
  - 使用者不能切到其他頁面啟動另一個 workflow

### Scenario 8: Job log 即時顯示且不重複

- **Given** Speed Pick workflow job 正在執行
- **When** frontend poll job status
- **Then**
  - RTDE/workflow log panel 顯示新增 log lines
  - 不重複 append 已顯示過的 log
  - job 完成、失敗、或 component unmount 時清理 polling interval

### Scenario 9: Good detection result 顯示成功狀態

- **Given** workflow job succeeded
- **And** job result 回傳 `detected=true` 且 `quality="GOOD"`
- **When** Speed Pick 顯示結果
- **Then**
  - 顯示 `result_image_url` 圖片
  - 顯示 `PICKED` 或等價 success 狀態
  - 顯示 matches、inliers、inlier ratio、reason
  - 顯示 estimated pick pose

### Scenario 10: Bad detection 不可顯示為成功 pick

- **Given** workflow job succeeded
- **And** job result 回傳 `detected=false` 或 `quality="BAD"`
- **When** Speed Pick 顯示結果
- **Then**
  - 若有 `result_image_url`，仍顯示結果圖片
  - final status 顯示 `DETECTION BAD` 或等價 failure/warning 狀態
  - 不得顯示為 `PICKED`
  - 顯示 `reason`

### Scenario 11: Job failed 後顯示錯誤並允許重試

- **Given** workflow job failed
- **When** poll job 回傳 `status="failed"`
- **Then**
  - 顯示 `error.code` 與 `error.message`
  - 停止 polling
  - 解鎖 page navigation
  - Run button 重新 enabled
  - 使用者可用同一 model/TCP retry

### Scenario 12: 無 pickable model 時顯示空狀態

- **Given** backend 沒有任何 built 且 pick-executable 的模型
- **When** 使用者進入 Speed Pick
- **Then**
  - 左側顯示 `No pickable model found. Build a model with pick pose first.`
  - 右側不顯示可執行 Run 的狀態
  - Run button disabled

### Scenario 13: Demo Mode 支援 Speed Pick

- **Given** `VITE_DEMO_MODE=1`
- **When** 使用者進入 Speed Pick 並按 Run
- **Then**
  - 使用 mock API 資料顯示 pickable models
  - `startPickWorkflow(... dry_run=false)` mock job 可完成
  - page 顯示 deterministic result image/status/logs
  - 行為與 real mode 的 UI state machine 一致

### Scenario 14: 1/4 + 3/4 workcell layout 顯示 result 不需捲動

- **Given** 使用者在一般 desktop viewport 進入 Speed Pick
- **When** 頁面 render 完成
- **Then**
  - 左側約 1/4 寬度顯示 `Choose model`
  - 右側約 3/4 寬度顯示 `Speed Pick Setup`
  - 右側同一個 workcell panel 內可看到 model info、`Pick TCP`、`Run`、`Detection result`、`RTDE log`
  - detection result 區域不應位在另一個需要往下捲動才看到的 panel

## 4. 測試情境 (Test Scenarios / Examples)

| ID | Scenario | Given | When | Then | Priority |
|---|---|---|---|---|---|
| TC1 | 顯示 Speed Pick nav | App loaded | 查看 top nav | `05 · Speed Pick` 出現在 `04 · Pick` 後方 | High |
| TC2 | 進入 Speed Pick | App loaded | 點 `05 · Speed Pick` | route 進入 Speed Pick page | High |
| TC3 | 模型清單 | backend 有 pickable models | 進入頁面 | 左側顯示 model rows，第一個自動選取 | High |
| TC4 | initial model | route payload 是 pickable model id | 進入頁面 | 該 model 被選取 | Medium |
| TC5 | 右側模型資訊 | 已選 model | render setup panel | 顯示 model id/counts/pick pose count/object image | High |
| TC6 | TCP selector | model 有多個 TCP | 展開 selector | options 等於 `pick_poses[].name` | High |
| TC7 | 單 TCP | model 只有一個 TCP | render selector | selector disabled 或 read-only | Medium |
| TC8 | 切換清結果 | 已有 result/error | 切換 model 或 TCP | 清除舊 result/error，不自動 run | High |
| TC9 | Run request | 已選 model/TCP | 按 Run | POST body 含 `dry_run=false` 和 selected `pick_name` | High |
| TC10 | Running control lock | job active | 操作 model/TCP/Run | controls disabled | High |
| TC11 | Running navigation lock | job active | 點 top nav/home/Open in 04 Pick | route 維持 Speed Pick，不能啟動其他 workflow | High |
| TC12 | Log 去重 | job poll 多次回傳重疊 logs | poll running job | log panel 不重複顯示同一行 | Medium |
| TC13 | Good result | job succeeded, quality GOOD | poll completes | 顯示 result image、PICKED、metrics、pose | High |
| TC14 | Bad result | job succeeded, quality BAD | poll completes | 顯示 DETECTION BAD，不顯示 picked success | High |
| TC15 | Failed job | job failed | poll completes | 顯示 error，navigation/Run 解鎖，可 retry | High |
| TC16 | Empty state | 無 pickable model | 進入頁面 | empty state，Run disabled | Medium |
| TC17 | Demo Mode | `VITE_DEMO_MODE=1` | 進入頁面並 Run | mock execute job 完成並顯示 deterministic result | Medium |
| TC18 | Workcell layout | desktop viewport | 進入 Speed Pick | left model rail + right 1:3 workcell; setup/result/log 同屏 | High |

## 5. 實作註記 (Implementation Notes)

### 5.1 Frontend route and navigation

Likely changes:

| File | Expected Change |
|---|---|
| `app/src/renderer/src/components/TopBar.tsx` | Add route label `05 · Speed Pick`; support disabled nav while Speed Pick job is active |
| `app/src/renderer/src/components/AppShell.tsx` | Add `speed-pick` route and guard route changes when Speed Pick reports busy |
| `app/src/renderer/src/screens/SpeedPickScreen.tsx` | New page for one-screen workflow |
| `app/src/renderer/src/screens/HomeScreen.tsx` | Optional entry card or link for Speed Pick |

Current route type only includes `home | create | viewer | pick`; implementation will need to extend route typing consistently.

### 5.2 Reuse existing API facade

Use existing API helpers where possible:

- `fetchPickableModels`
- `fetchModelCaptures`
- `startPickWorkflow`
- `pollJob`

`StartPickWorkflowParams` already supports `dry_run` and `skip_observe_move`. Speed Pick should send `dry_run=false`. It does not need a new backend endpoint for the first implementation.

### 5.3 Page state machine

Suggested local states:

- `ready`
- `running`
- `picked`
- `detection_bad`
- `failed`

State should be derived from:

- `activeJobId`
- `job.status`
- `job.result.detected`
- `job.result.quality`
- `job.error`

### 5.4 Navigation lock implementation

Speed Pick should report busy state to `AppShell`, for example:

```tsx
<SpeedPickScreen
  onExit={go}
  initialModel={route.payload}
  onBusyChange={setRouteLocked}
/>
```

`AppShell.go(...)` should ignore route changes while route is locked. `TopBar` should also visually disable navigation buttons while locked.

The route lock must release when:

- job succeeds
- job fails
- Speed Pick component cleanup runs after job is no longer active

### 5.5 Result image rule

Render backend `result_image_url` as a plain image:

```tsx
<img src={resolveAssetUrl(result.result_image_url) ?? ''} alt="Speed pick detection result" />
```

Do not render frontend SVG/canvas/TCP marker overlays on top of the result image.

### 5.6 Demo Mode

Current mock API already supports `startPickWorkflow(params)` and creates `pick_execute` jobs when `dry_run=false`. Speed Pick should go through the same API facade as real mode so Demo Mode stays automatic.

### 5.7 Safety note

Speed Pick intentionally skips the manual confirm step from 04 Pick. Because this can move the real robot after one button press:

- The Run button should be visually distinct.
- Running status should be prominent.
- Page navigation lock is required while active.
- Emergency stop API is out of scope for this feature unless added by a separate requirement.

## 6. 補充說明 (Additional Notes)

### Related documents

- `documents/implements/speed_pick_page_design.md`
- `documents/implements/F01-PickPage.md`
- `documents/implements/F05-PickPage-DetectTcpSelector.md`
- `documents/implements/F06-Frontend-DemoMode.md`
- `documents/modules/GUI設計規劃.md`
- `documents/modules/frontend_demo_mode.md`

### Out of Scope

- Removing 04 Pick
- Adding TCP teaching/editing to Speed Pick
- Creating models from Speed Pick
- New backend pick workflow endpoint
- New emergency stop backend API
- Frontend overlays on detect result image
- A manual confirm step inside Speed Pick

### Documentation follow-up after implementation

- Update this F09 document with implementation record, changed files, tests run, and known limitations.
- Update `documents/modules/GUI設計規劃.md` to include `05 Speed Pick`.
- Update `documents/modules/frontend_demo_mode.md` if Demo Mode behavior changes or receives new test coverage.

## 7. Implementation Record

### Status

Implemented.

### Implementation Summary

- Added `SpeedPickScreen` as a one-screen workflow for selecting a pickable model, selecting a pick TCP, and running the complete pick workflow with `dry_run=false`.
- Reorganized Speed Pick into a 1:3 layout: left model selection rail, right single workcell panel containing model info, TCP selector, Run, detection result, and compact RTDE log.
- Added `05 · Speed Pick` to the top navigation and `speed-pick` route handling in `AppShell`.
- Added route locking: while a Speed Pick job is active, top navigation, brand/home navigation, `Open in 04 Pick`, model selection, TCP selection, and Run are disabled or ignored.
- Reused existing API facade helpers: `fetchPickableModels`, `fetchModelCaptures`, `startPickWorkflow`, and `pollJob`.
- Rendered result images via `resolveAssetUrl()` as plain `<img>` elements with no frontend overlays.
- Added status handling for `PICKED`, `DETECTION BAD`, and `FAILED`.
- Added Demo Mode coverage through the existing mock API execute-job path.

### Test Coverage

| Test File | Coverage |
|---|---|
| `app/src/renderer/src/screens/__tests__/SpeedPickScreen.test.tsx` | F09 TC1-TC18 route, setup, 1:3 workcell layout, model/TCP selection, Run request, active-job locks, log dedupe, good/bad/failed result states, empty state, and Demo Mode |

### Files Changed

#### Production Files

- `app/src/renderer/src/screens/SpeedPickScreen.tsx`
- `app/src/renderer/src/components/AppShell.tsx`
- `app/src/renderer/src/components/TopBar.tsx`
- `app/src/renderer/src/screens/HomeScreen.tsx`

#### Test Files

- `app/src/renderer/src/screens/__tests__/SpeedPickScreen.test.tsx`

#### Documentation Files

- `documents/implements/F09-SpeedPickPage.md`
- `documents/modules/GUI設計規劃.md`
- `documents/modules/frontend_demo_mode.md`

### Acceptance Criteria Verification

| Acceptance Criterion | Status | Evidence |
|---|---|---|
| Scenario 1: 顯示 05 Speed Pick 頁簽 | Passed | `TC1/TC2: shows 05 Speed Pick in top navigation and opens the page` |
| Scenario 2: 左側載入可 pick 模型 | Passed | `TC3/TC5: loads pickable models and shows selected model details` |
| Scenario 3: 支援 initial model payload | Passed | `TC4: selects initial model when provided` |
| Scenario 4: 右側顯示模型資訊與 TCP selector | Passed | `TC3/TC5`, `TC6/TC7` |
| Scenario 5: 切換模型或 TCP 會清掉舊結果 | Passed | `TC8/TC9/TC13` |
| Scenario 6: Run 啟動完整 pick workflow | Passed | `TC8/TC9/TC13` verifies `dry_run=false` request body |
| Scenario 7: Job 執行中鎖定頁面操作 | Passed | `TC10`, `TC11` |
| Scenario 8: Job log 即時顯示且不重複 | Passed | `TC12` |
| Scenario 9: Good detection result 顯示成功狀態 | Passed | `TC8/TC9/TC13` |
| Scenario 10: Bad detection 不可顯示為成功 pick | Passed | `TC14` |
| Scenario 11: Job failed 後顯示錯誤並允許重試 | Passed | `TC15` |
| Scenario 12: 無 pickable model 時顯示空狀態 | Passed | `TC16` |
| Scenario 13: Demo Mode 支援 Speed Pick | Passed | `TC17` |
| Scenario 14: 1/4 + 3/4 workcell layout 顯示 result 不需捲動 | Passed | `TC18` |

### FXX Test Case Verification

| FXX Test Case | Status | Automated Test Evidence |
|---|---|---|
| TC1 | Passed | `TC1/TC2: shows 05 Speed Pick in top navigation and opens the page` |
| TC2 | Passed | `TC1/TC2: shows 05 Speed Pick in top navigation and opens the page` |
| TC3 | Passed | `TC3/TC5: loads pickable models and shows selected model details` |
| TC4 | Passed | `TC4: selects initial model when provided` |
| TC5 | Passed | `TC3/TC5: loads pickable models and shows selected model details` |
| TC6 | Passed | `TC6/TC7: lists TCP options and disables selector for a single-TCP model` |
| TC7 | Passed | `TC6/TC7: lists TCP options and disables selector for a single-TCP model` |
| TC8 | Passed | `TC8/TC9/TC13: clears old result on TCP change, runs dry_run=false, and shows good result` |
| TC9 | Passed | `TC8/TC9/TC13: clears old result on TCP change, runs dry_run=false, and shows good result` |
| TC10 | Passed | `TC10: disables model, TCP, and Run while a job is active` |
| TC11 | Passed | `TC11: blocks top navigation while Speed Pick job is active` |
| TC12 | Passed | `TC12: deduplicates overlapping job logs` |
| TC13 | Passed | `TC8/TC9/TC13: clears old result on TCP change, runs dry_run=false, and shows good result` |
| TC14 | Passed | `TC14: shows DETECTION BAD instead of PICKED when detection quality is bad` |
| TC15 | Passed | `TC15: shows error and enables retry when job fails` |
| TC16 | Passed | `TC16: shows empty state and disables Run when no pickable models exist` |
| TC17 | Passed | `TC17: runs Speed Pick through the mock API facade in Demo Mode` |
| TC18 | Passed | `TC18: uses a 1:3 workcell layout with setup and result in the right workspace` |

### Commands Run

```bash
cd app && npm test -- SpeedPickScreen.test.tsx
cd app && npm test
cd app && npm run build
git diff --check
```

### Result

- Red phase observed: `npm test -- SpeedPickScreen.test.tsx` failed because `@/screens/SpeedPickScreen` did not exist.
- Targeted F09 suite passed: 13 tests passed.
- Full frontend suite passed: 8 test files, 130 tests passed.
- Production build passed with `electron-vite build`.
- `git diff --check` passed.

### Assumptions

- Speed Pick uses the existing backend `POST /api/pick/workflow` endpoint with `dry_run=false`; no backend endpoint change is required.
- `approach_height_m` remains fixed at `0.05` for the first implementation, matching F09.
- The top navigation lock is app-local UI locking; it does not prevent external OS/window actions.

### Deferred Items

- No new emergency stop API was added; this remains out of scope.
- Home page Speed Pick card/link was not added because F09 marked it optional and top navigation provides access.
- Shared component extraction with `PickScreen` was deferred; current implementation keeps changes localized and tests cover the behavior.

## 附錄：TDD 需求實作流程提醒 (TDD Implementation Checklist)

請依照以下流程進行開發：

1. 撰寫 Speed Pick route/nav tests，確認 `05 · Speed Pick` 可進入。
2. 撰寫 model list、TCP selector、Run request tests。
3. 撰寫 active job control lock 與 navigation lock tests。
4. 撰寫 good result、bad result、failed job tests。
5. 實作最小 Speed Pick page。
6. 重構共用 Pick model list、polling、result panel helper時，保持 F09 tests 綠燈。
7. 更新本文件與 module documents 的 implementation record。
