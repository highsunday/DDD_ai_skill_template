---
name: ddd-queue
description: "DDD 長時間工作佇列技能。建立或執行 documents/queue/ 下的 QXX queue 文件，適用於 2 到 5 個已排序、可自動推進的功能、Bug 修正或重構工作；建立 queue 時必須先用集中式 grill-me 釐清所有 item 的需求、設計問題、依賴、驗收與停止條件，更新 QXX 後才允許逐項執行。每個 item 必須由新的 Codex 或 Claude Code session 單獨處理，完整執行 ddd-start、ddd-doc、ddd-tdd，完成後 git commit，並透過 Agent Communication Ledger 記錄兩個 agent、orchestrator 與使用者之間的派工、問題、回答、決策、測試與接棒紀錄。遇到 blocker 則寄信通知使用者並停止；整批全部完成時寄完成通知。queue worker 內部呼叫的 ddd-tdd 不寄單項完成通知。不要用於單一工作、需求仍需人類逐階段決策、或後續階段範疇必須等前一階段完成後才能定義的工作，這些分別使用 ddd-doc/ddd-tdd 或 ddd-plan。"
---

# DDD Queue

管理一批已排序、可自動推進的 DDD 工作，目標是延長 AI 可自主工作的時間，減少使用者被逐項打斷。`ddd-queue` 可以處理獨立 item，也可以處理明確相依的連續 item；`ddd-plan` 則處理尚需要規劃、拆解或人類審查的大型改動。

判斷方式：

- 使用 `ddd-queue`：使用者已能列出 2 到 5 個具體 item，且每個 item 都有需求、驗收方式、停止條件；若有相依，依賴與解鎖條件能在文件、commit 或測試結果中被確認。
- 使用 `ddd-plan`：後續 item 的範疇要等前一階段完成才知道、存在架構路線選擇、或階段拆分本身需要人類審查。

## 核心規則

1. 每次只讓一個新 session 處理一個 queue item。
2. 不在同一個 Codex / Claude Code session 連續實作多個 item。
3. 每完成一個 item 必須建立一個 git commit，commit 內包含程式碼、測試、F/R/B 文檔與 queue 狀態更新。
4. 一次執行最多 5 個 item；未指定時預設最多 3 個。
5. 遇到 blocker 立即停止整個 queue，不繼續下一個 item。
6. 自動批准只適用於清楚、低風險、可驗收的 item；模糊或高風險 item 必須 blocked。
7. 執行 queue 前工作樹必須乾淨，避免自動 commit 夾帶使用者未提交變更。
8. 相依 item 必須宣告 `depends_on` 與 `unlock_condition`；解鎖條件無法驗證時必須 blocked。
9. 每個 worker 子 session 都必須把指定 item 當成一個新的 DDD 工作，完整執行 `ddd-start → ddd-doc → ddd-tdd`，不得直接改程式碼。
10. 子 session 不進行即時對話；需要使用者回答時，寫入 queue 的 blocked / questions 欄位，orchestrator 通知使用者並停止。
11. Queue 文件是 Codex、Claude Code、orchestrator 與使用者的共享通訊帳本；所有跨 session 訊息都必須追加到 `Agent Communication Ledger`。
12. 通訊紀錄採 append-only：不得刪改既有 log entry；若先前紀錄錯誤，追加 correction / superseded entry。
13. 建立 queue 時必須先集中執行 `grill-me`，一次性釐清所有 item 的需求、設計風險、依賴、驗收與停止條件。
14. Queue 未完成集中釐清前不得執行 worker；執行時必須確認 `intake_grill_status: completed` 且每個要執行的 item 都是 `clarification_status: clarified`。
15. QXX 主文件只保留執行所需摘要、索引與未解決問題；長 stdout、完整對話與歷史細節歸檔到 `documents/queue/logs/`，避免後續 worker 重複消耗 token。
16. 若 queue 要在 blocked 或整批完成時寄信，QXX frontmatter 或 `documents/ddd-email-notify.md` 必須明確設定 `notify_email_from` 與 `notify_email_to`；寄信與設定驗證交給 `ddd-email-notify` 規則處理，不得把 email 密碼、token、SMTP key 或 app password 寫入 QXX。
17. queue worker 內部呼叫的 `ddd-tdd` 不寄單項完成通知；只有 queue orchestrator 在整批全部完成時寄一次 completed 通知。

## Queue 文件位置

queue 文件放在專案根目錄：

```text
documents/queue/
├── Q00-queue-template.md
└── Q01-<batch-name>.md
```

若 `documents/queue/Q00-queue-template.md` 不存在，從 `ddd-create-folder/templates/Q00-queue-template.md` 複製，或依照本技能的格式建立。

## Queue Item 格式

每個 item 必須有以下欄位：

```md
## Q01-01 <功能名稱>

id: Q01-01
type: FXX
agent: codex
status: pending
clarification_status: pending
depends_on: []
unlock_condition: 無
auto_approve: true
commit_required: true
implemented_doc: —
commit: —
worker_session: —
worker_log: —
handoff_summary: —
communication_entries: []

### 需求
<清楚描述要完成的使用者行為>

### 驗收方式
- [ ] <使用者可操作的確認動作與預期結果>

### 停止條件
- <何時應 blocked 而不是自行猜測>

### 阻塞記錄
尚無。
```

欄位規則：

- `type`: `FXX` / `BXX` / `RXX`
- `agent`: `codex` / `claude` / `auto`
- `status`: `pending` / `in_progress` / `completed` / `blocked` / `skipped`
- `clarification_status`: `pending` / `clarifying` / `clarified` / `blocked`；只有 `clarified` 可以進入 worker
- `depends_on`: 依賴的 item id 清單，例如 `[Q01-01]`；無依賴時填 `[]`
- `unlock_condition`: 何時可以開始此 item，例如 `Q01-01 completed 且登入頁已可正常提交`
- `auto_approve`: `true` 表示可由 worker 自行建立 F/R/B 文檔並視為已核准；但 worker 仍要在文檔清楚後才開始改程式碼。
- `implemented_doc`: 完成後填入實際文件路徑，例如 `documents/implements/F03-user-search.md`
- `commit`: 完成後填入 commit hash
- `worker_session`: orchestrator 啟動子 session 時可填入 `codex` / `claude` 與時間戳，方便追蹤
- `worker_log`: worker 完成或 blocked 時填入摘要、測試命令、風險與限制
- `handoff_summary`: 給下一個 agent 的短接棒摘要，包含目前狀態、重要檔案、決策與風險
- `communication_entries`: 關聯到此 item 的 `Agent Communication Ledger` entry id 清單，例如 `[L001, L004]`

## Email Notification Settings

QXX 支援 blocked 與整批 completed 通知信。QXX frontmatter 可覆蓋 project config 的寄信設定：

```yaml
notify_email_from: <可選，寄信來源；必須是目前環境已授權可寄出的信箱>
notify_email_to: <可選，通知收件信箱>
notify_on_queue_blocked: true
notify_on_queue_completed: true
notify_email: <deprecated，可選；舊版阻塞通知收件信箱，請改用 notify_email_to>
```

設定規則：

1. `notify_email_from` 是寄信來源，不是密碼或 provider 設定。
2. `notify_email_to` 是 blocked 或 completed 時要通知的收件信箱。
3. `notify_email` 僅作為舊文件相容欄位；若 QXX 只有 `notify_email`，orchestrator 視為 `notify_email_to`，並在下次更新 QXX 時補上 `notify_email_to`。
4. 若使用者要求 queue 通知，建立或更新 QXX 時必須詢問並填入 `notify_email_from` 與 `notify_email_to`，或確認 `documents/ddd-email-notify.md` 已有可用設定。
5. 若使用者沒有要求寄信，兩個欄位可保留空值；blocked 時直接在目前對話回報。
6. 若寄信工具不支援指定 From，例如只能用目前已登入帳號寄出，orchestrator 必須確認工具的授權帳號與 `notify_email_from` 一致；不一致時不得寄信，必須停止 queue 並在目前對話回報。
7. QXX 不保存任何 email credential。寄信能力由目前 Codex / Claude / MCP / CLI 執行環境提供。
8. 設定、驗證與寄信細節依 `ddd-email-notify` 執行。
9. 每次寄信或寄信失敗都必須追加 ledger entry，建議使用 `notification` 或 `blocked` type。
10. 達到 `batch_limit` 後停止但仍有 pending item 時，不算 queue completed，不寄 completed 通知。

## 集中需求釐清（Queue Intake Grill）

`ddd-queue` 的核心目標是讓使用者在執行前一次性處理主要不確定性，避免每個 worker 子 session 都反覆打斷使用者。

建立或更新 QXX 時，先執行集中式 `grill-me`：

1. 將整批 queue item 視為同一個 intake session，而不是逐項分開問。
2. 逐項檢查：
   - 使用者目標與可觀察行為
   - 驗收方式是否能由人操作確認
   - 自動化測試是否可推導
   - item 間依賴與 `unlock_condition`
   - UI / API / 資料模型 / 權限 / migration / 安全風險
   - 非目標與停止條件
   - 哪些問題若不回答會讓 worker blocked
3. 將問題集中整理成「Queue Intake Questions」，用 item 分組，一次問使用者。
4. 使用者回答後，更新 QXX：
   - frontmatter `intake_grill_status: completed`
   - frontmatter `ready_for_execution: true`
   - 每個可執行 item `clarification_status: clarified`
   - 每個 item 的需求、驗收方式、停止條件、依賴、解鎖條件與設計註記
   - `Agent Communication Ledger` 追加 `intake-question`、`answer`、`decision` entry
5. 若仍有未回答的阻塞問題，queue 保持 `ready_for_execution: false`，對應 item 設為 `clarification_status: blocked`，不要啟動 worker。

集中 grill-me 的輸出必須寫入 QXX 的「Queue Intake Review」區塊。建議包含：

```md
## Queue Intake Review

intake_grill_status: completed
ready_for_execution: true
reviewed_by: codex
reviewed_at: <YYYY-MM-DD HH:MM>

### Clarification Matrix

| Item | Clarification | Key Decisions | Remaining Questions |
|------|---------------|---------------|---------------------|
| Q01-01 | clarified | <決策摘要> | — |

### Queue Intake Questions

#### Q01-01
- Q: <問題>
- A: <使用者回答>

### Cross-item Decisions

- <跨 item 的依賴、順序、共用設計決策>
```

worker 子 session 不應對 `clarification_status: clarified` 的 item 再次執行互動式 grill-me。若 worker 在實作期間發現 QXX 未涵蓋的新矛盾，必須 blocked，追加 `question` entry，讓 orchestrator 停止 queue 並請使用者回答。

## Context Budget / Token Policy

`ddd-queue` 的預設策略是「主文件短、歷史可查、worker 精讀」。QXX 是工作狀態與索引，不是完整逐字稿。

主文件保留：

- frontmatter、總覽、Queue Intake Review 摘要
- 每個 item 的需求、驗收、停止條件、目前狀態
- 每個 item 最多 8 行的 `handoff_summary`
- Agent Communication Ledger 的 `Log Index`
- 最近、未解決、或下一個 worker 必須讀的 active entries

主文件不保留：

- 完整 stdout / stderr
- 完整測試輸出
- 長篇 worker 回覆
- 已完成且不再影響後續 item 的歷史細節

歸檔規則：

1. 長內容放到 `documents/queue/logs/<QXX>-<entry-id>.md` 或 `documents/queue/logs/<QXX>-archive.md`。
2. QXX 的 `Log Index` 保留 `Archive Ref` 路徑。
3. 單一 ledger entry 建議不超過 1200 字；超過時保留摘要，全文歸檔。
4. 單一 `worker_log` 建議不超過 8 行；測試輸出只列命令、結果與關鍵錯誤。
5. 單一 `handoff_summary` 建議不超過 8 行；只寫下一個 agent 必須知道的狀態、檔案、決策、測試與風險。
6. 當 QXX 超過約 500 行、ledger 超過 20 筆、或已完成 item 超過 3 筆 entry 時，先做 compaction：保留索引、狀態、決策、未解問題與 handoff，將舊細節歸檔。

worker 讀取上下文時，只讀：

- QXX frontmatter、總覽與 Queue Intake Review
- 指定 item 區塊
- 依賴 item 的 `handoff_summary`、`implemented_doc`、`commit`
- `Log Index`
- 指定 item 與依賴 item 在 `communication_entries` 中列出的 active entries

除非遇到 blocker、矛盾或驗收需要，worker 不讀完整 archive。

## Agent Communication Ledger

每份 QXX 必須有全域 `Agent Communication Ledger`。這是兩個 agent 之間溝通、orchestrator 派工、使用者回答問題，以及事後追蹤決策的唯一來源。

Ledger 原則：

1. **Append-only**：只能追加新 entry，不刪除、不覆寫既有 entry。
2. **可讀優先**：記錄可被使用者事後閱讀的操作摘要、問題、答案、決策、測試證據與接棒資訊。
3. **不記錄隱藏推理**：不要保存私有推理草稿；保存可驗證的判斷、假設、依據與結論。
4. **雙向訊息都記錄**：orchestrator → worker 的派工、worker → orchestrator 的狀態、worker → user 的問題、user → worker 的回答，都要有 entry。
5. **長輸出用摘要**：stdout / 測試輸出太長時，記摘要與關鍵錯誤；若有外部 log 檔，再記路徑。
6. **索引先行**：後續 agent 先讀 `Log Index`，只打開與當前 item、依賴 item 或 unresolved question 相關的 entry / archive。

建議格式：

```md
## Agent Communication Ledger (Append-only)

### Log Index

| Entry | Time | Item | From -> To | Type | Summary | Archive Ref |
|-------|------|------|------------|------|---------|-------------|
| L001 | 2026-05-23 10:00 | Q01-01 | orchestrator -> codex | dispatch | 指派 Q01-01 | — |

### Active Entries

#### L001 — 2026-05-23 10:00 — Q01-01 — orchestrator -> codex — dispatch

**Message**
指派 Q01-01。請以 ddd-start -> ddd-doc -> ddd-tdd 處理，完成後 commit。

**Context**
- Queue file: `documents/queue/Q01-example.md`
- Depends on: []
- Unlock condition: 無

**Expected Response**
- 更新 item 狀態
- 建立 F/R/B 文檔
- 記錄紅燈與綠燈測試
- commit hash

**Artifacts**
- —

**Follow-up**
- —

**Archive**
- —
```

Entry `Type` 建議使用：

- `dispatch`: orchestrator 指派 worker
- `intake-question`: 集中 grill-me 階段提出的整批需求釐清問題
- `status`: worker 回報目前進度
- `ddd-start`: worker 完成需求類型判斷
- `ddd-doc`: worker 建立或更新 F/R/B 文檔
- `tdd-red`: worker 記錄紅燈測試
- `tdd-green`: worker 記錄綠燈與驗收
- `question`: worker 需要使用者回答
- `answer`: 使用者或 orchestrator 回答
- `decision`: 記錄已採納的產品或技術決策
- `handoff`: 給下一個 agent 的接棒摘要
- `blocked`: worker blocked 並停止
- `notification`: orchestrator 已寄出 blocked 通知，或寄信失敗後改為對話回報
- `completed`: worker 完成 item
- `correction`: 修正先前 entry
- `archive`: 將舊 entry 或長輸出移到 archive
- `compaction`: 壓縮 QXX 主文件，保留索引、狀態與 handoff

## 建立 Queue

當使用者要求排一批工作、延長 AI 自主工作時間、或減少逐項打斷時：

1. 讀取 `CONTEXT.md`，使用既有領域語言。
2. 建立 `documents/queue/`，並確認下一個可用的 `QXX` 編號。
3. 先建立 QXX 草稿，frontmatter 設為：
   - `status: draft`
   - `intake_grill_status: pending`
   - `ready_for_execution: false`
4. 將每個需求整理成 item 草稿；每個 item 初始 `clarification_status: pending`。
5. 執行集中式 `grill-me`，一次性釐清所有 item。
6. 使用者回答後，完整更新 QXX 的 Queue Intake Review、item 需求、驗收、停止條件、依賴與設計註記。
7. 若 item 之間有依賴，寫入 `depends_on` 與 `unlock_condition`，並依依賴順序排序。
8. 若後續 item 的需求要等前一 item 完成後才知道，停止並建議改用 `ddd-plan`。
9. 若 item 太模糊，將該 item 標成 `clarification_status: blocked`，不要放進可執行狀態。
10. 在 queue frontmatter 填入：
   - `status: ready`（僅當所有待執行 item 都 clarified）
   - `intake_grill_status: completed`
   - `ready_for_execution: true`
   - `batch_limit: 3`，除非使用者指定 1 到 5
   - `notify_email_from`，若使用者要求 queue 通知
   - `notify_email_to`，若使用者要求 queue 通知
   - `notify_on_queue_blocked: true`
   - `notify_on_queue_completed: true`
   - `notify_email`，僅在遷移舊 QXX 時作為相容別名，不要在新文件主動新增

完成後回報 queue 文件路徑與 item 清單。若使用者同時要求執行，繼續「執行 Queue」。

## 執行 Queue

AI 目前所在 session 是 orchestrator。orchestrator 只負責挑選 item、啟動新的子 session、檢查結果、寄信與停止；不要在 orchestrator session 直接實作功能。

### 子 Session 溝通模型

使用 `codex exec` 或 `claude -p` 啟動的 worker 是非互動子 session。orchestrator 可以讀取它的 stdout / stderr 與最終結果，但不應假設能在執行中可靠地插入新問題或修正方向。

預設溝通方式是文件協議：

1. orchestrator 啟動 worker 前，先追加一筆 `dispatch` entry，並將 entry id 寫入 item 的 `communication_entries`。
2. worker 開始後，先讀取 `Log Index`、指定 item 的 active `communication_entries`、所有依賴 item 的 `handoff_summary`。
3. worker 完成 ddd-start / ddd-doc / ddd-tdd 重要節點時，追加對應 entry。
4. worker 需要人類決策時，將 item 設為 `status: blocked`，追加 `question` 或 `blocked` entry，並在 item 中填入 `blocker_reason`、`questions` 或 `need_user_decision`。
5. orchestrator 偵測 blocked 後寄信或回報使用者，並停止 queue。
6. 使用者補充答案後，orchestrator 追加 `answer` entry，再啟動新的 worker session 繼續。

若真的需要即時雙向溝通，應改用互動式 tmux / remote-control 類型的 runner；但這會降低 queue 的可重現性，不作為預設流程。

### 步驟 1 — 前置檢查

1. 讀取 queue 文件。
2. 確認集中釐清已完成：`ready_for_execution: true`、`intake_grill_status: completed`，且本輪要執行的 item 都是 `clarification_status: clarified`。
3. 若集中釐清未完成，停止執行並回到「集中需求釐清（Queue Intake Grill）」；不要啟動 worker。
4. 確認 `git status --short` 為乾淨。若不乾淨，停止並請使用者先處理；不要自動 commit 使用者既有變更。
5. 若 QXX 設定了 `notify_email_from`、`notify_email_to` 或舊欄位 `notify_email`，依 `ddd-email-notify` 驗證 email notification 設定：
   - 若只有舊欄位 `notify_email`，將其視為 `notify_email_to`，並在下一次 QXX 更新補上新欄位。
   - 若使用者要求寄信但缺少 `notify_email_from` 或 `notify_email_to`，停止並要求補齊。
   - 確認目前環境有可用寄信工具；若沒有，執行可以繼續，但 blocked 時只能在目前對話回報。
   - 若寄信工具不支援指定 From，確認目前授權寄件帳號與 `notify_email_from` 一致；不一致時停止並要求使用者更正寄件來源或切換授權帳號。
6. 確認 `codex` / `claude` CLI 是否存在：
   - Codex: `codex exec`
   - Claude Code: `claude -p`
7. 決定本輪最多處理幾個 item：使用使用者指定值，否則使用 queue 的 `batch_limit`，再否則預設 3；絕不可超過 5。

### 步驟 2 — 選擇下一個 item

依文件順序選擇第一個 `status: pending` 且依賴已完成的 item。

依賴檢查：

1. 讀取 item 的 `depends_on`。
2. 確認每個依賴 item 都是 `status: completed`，且 `commit` 不為空。
3. 檢查 `unlock_condition` 是否能從 queue、F/R/B 文檔、測試結果或當前程式碼狀態合理確認。
4. 若依賴 item 尚未完成，先處理最靠前的未完成依賴。
5. 若依賴 item blocked、缺少 commit，或解鎖條件不可驗證，將目前 item blocked 並停止 queue。

若 `agent: auto`：
- 若兩個 CLI 都可用，優先沿用 queue 中前一個 item 的交替策略。
- 若只有一個 CLI 可用，使用可用的 CLI。
- 若沒有 CLI 可用，將 item blocked 並停止。

### 步驟 3 — 啟動新的 worker session

每個 item 都必須用新的 CLI process。不可使用 `codex resume`、`claude --continue` 或 `claude --resume`。

Codex worker 範例：

```bash
codex exec \
  --cd "$REPO_ROOT" \
  --sandbox danger-full-access \
  --ask-for-approval never \
  --ephemeral \
  "$WORKER_PROMPT"
```

Claude Code worker 範例：

```bash
claude -p \
  --no-session-persistence \
  --permission-mode bypassPermissions \
  "$WORKER_PROMPT"
```

若在 Claude Code 中需要明確工作目錄，先讓 shell command 的工作目錄設為專案根目錄，再執行 `claude -p`。

### Worker Prompt

給子 session 的 prompt 必須自包含，至少包含以下內容：

```text
你是 DDD queue worker。只處理一個 queue item，不要處理其他 item。

Repo: <專案根目錄>
Queue file: <queue 文件路徑>
Item id: <QXX-YY>

必須遵守：
1. 讀取 ddd-queue 規則；若技能不可用，依本 prompt 執行。
2. 讀取 queue 文件，只處理指定 item。
3. 確認 queue 已完成集中釐清：ready_for_execution: true、intake_grill_status: completed、指定 item clarification_status: clarified。若否，blocked，不要實作。
4. 開始前確認工作樹乾淨；若不乾淨，回報 blocked，不要 commit。
5. 檢查 depends_on；所有依賴 item 必須 completed 且有 commit，unlock_condition 必須可驗證。
6. 閱讀 Queue Intake Review、Log Index、指定 item 的 active communication_entries、依賴 item 的 handoff_summary、implemented_doc、commit 摘要與測試記錄，建立本 item 所需上下文；除非 blocked 或上下文矛盾，不讀完整 archive。
7. 將指定 item 標成 in_progress，填入 worker_session，並追加 status entry。
8. 以指定 item 作為新工作執行 ddd-start：判斷是 FXX / BXX / RXX，並載入 CONTEXT.md；若 ddd-start 會要求 grill-me，先對照 Queue Intake Review，已回答則繼續，未回答才 blocked；追加 ddd-start entry。
9. 執行 ddd-doc：建立或更新對應 FXX / BXX / RXX 文檔，內容必須包含需求、驗收標準、測試場景、停止條件與假設；追加 ddd-doc entry。
10. 若 auto_approve: true 且 ddd-doc 產出的文檔清楚、低風險、可測試，將該文檔視為本 item 的已核准文檔；否則 blocked，追加 question / blocked entry，等待使用者審核。
11. 執行 ddd-tdd：先寫失敗測試並確認紅燈，追加 tdd-red entry；再實作最小綠燈，最後驗收並更新文檔實作記錄，追加 tdd-green entry。因本次 ddd-tdd 是由 ddd-queue worker 呼叫，必須抑制 ddd-tdd 單項完成通知，不得寄信。
12. 不得跳過紅燈測試；若無法建立合理失敗測試，blocked 並說明原因。
13. 若需求模糊、高風險、測試無法穩定通過、需要使用者決策，將 item 標成 blocked，填寫 blocker_reason、questions 或 need_user_decision，追加 question / blocked entry，停止。
14. 完成時更新 queue item：status: completed、implemented_doc、commit、worker_log、handoff_summary、communication_entries，並追加 handoff / completed entry。
15. git commit 必須只包含本 item 相關檔案；使用明確 git add 檔案清單，不要使用 git add .
16. commit 後確認工作樹乾淨。
17. 若 QXX 超過 context budget，先做 compaction，將長 log 歸檔並追加 compaction / archive entry。
18. 最終回覆只回報 item 狀態、commit hash、執行測試命令、ledger entries、ddd-tdd completed notification suppressed by queue 與任何限制。不要開始下一個 item。
```

## Worker DDD 流程

worker 子 session 必須按以下順序執行，缺一不可：

1. **ddd-start**：將 queue item 當成使用者的新工作，判斷類型、載入 `CONTEXT.md`、檢查是否需要 grill / zoom-out / diagnose 等前置流程。若需求不適合自動批准，blocked。
2. **ddd-doc**：建立或更新一份 FXX / BXX / RXX 文檔。queue item 不是實作文檔，只是派工來源；真正的需求權威仍是 F/R/B 文檔。
3. **ddd-tdd**：以剛建立的 F/R/B 文檔作為唯一需求來源，執行紅燈、綠燈、驗收、文檔更新；因為這是 queue worker 內部執行，不寄 ddd-tdd 單項完成通知。
4. **queue closeout**：更新 queue item 狀態、關聯文檔、測試命令、commit hash 與 worker log。
5. **communication closeout**：追加 `handoff` / `completed` entry，更新 item 的 `handoff_summary` 與 `communication_entries`。

若 worker 發現自己已經開始修改生產代碼但尚未建立 F/R/B 文檔，必須停止、回復本 item 未完成變更，先回到 ddd-doc。

## Worker 完成規則

worker 完成 item 前必須：

1. 已在 worker log 中記錄 ddd-start 判斷結果。
2. 建立或更新一份 F/R/B 文檔，並將路徑寫入 `implemented_doc`。
3. 依該文檔新增或更新測試。
4. 觀察並記錄至少一個正確原因的紅燈測試。
5. 執行相關測試，必要時執行更廣的回歸測試。
6. 更新 F/R/B 文檔的實作記錄。
7. 更新 queue item 狀態、總覽表與 worker log。
8. 更新 `handoff_summary`，讓下一個 agent 不必重新讀完整 stdout。
9. 追加必要的 ledger entry：至少包含 dispatch、ddd-start、ddd-doc、tdd-red、tdd-green、handoff 或 blocked；長內容只保留摘要並寫入 archive ref。
10. 建立 git commit。
11. 確認 `git status --short` 乾淨。
12. 若觸發 context budget 門檻，執行 compaction，確保下一個 worker 不需要讀完整歷史。

建議 commit message：

```text
feat: complete Q01-01 <功能名稱>
fix: complete Q01-02 <修正名稱>
refactor: complete Q01-03 <重構名稱>
```

## Blocker 規則

以下情況必須 blocked：

- 需求缺少可驗收行為。
- queue 未完成集中 grill-me，或 item `clarification_status` 不是 `clarified`。
- item 的依賴未宣告、依賴 item blocked、依賴 commit 缺失，或 `unlock_condition` 無法驗證。
- 需要修改資料模型、權限、付款、安全、登入、隱私、資料刪除或 migration，但 queue 未明確授權。
- 連續兩次測試失敗後仍無法辨識根因。
- 需要使用者選擇產品行為或 trade-off。
- 工作樹不乾淨，可能夾帶非本 item 變更。
- 找不到指定的頁面、模組、API 或測試入口，且無法從 repo 合理推導。

blocked 時更新 item：

```md
status: blocked
blocker_reason: <具體原因>
questions:
- <需要使用者回答的問題>
need_user_decision:
- A: <選項>
- B: <選項>
worker_log: <已完成哪些步驟、卡在哪一步、是否有未提交變更>
communication_entries: [<相關 LXXX>]
```

不要 commit 不完整的生產代碼。若已產生部分變更且無法安全完成，留下清楚記錄並停止；orchestrator 不得繼續下一個 item。

## 通知使用者

orchestrator 偵測到 blocked 後：

1. 停止 queue。
2. 依 `ddd-email-notify` 讀取 queue frontmatter 的 `notify_email_from` 與 `notify_email_to`。
3. 若舊文件只有 `notify_email`，將其視為 `notify_email_to`，並在 QXX 更新時補上 `notify_email_to`。
4. 若 `notify_on_queue_blocked: false`，不寄信，直接在目前對話回報 blocked。
5. 若沒有 `notify_email_to`、沒有可用寄信工具、或寄信來源無法驗證，直接在目前對話回報 blocked。
6. 若寄信工具不支援指定 From，必須確認目前授權帳號與 `notify_email_from` 一致；若不一致，不寄信並在目前對話回報。
7. 寄信成功或失敗都追加 `notification` ledger entry，記錄 From、To、subject、delivery status 與是否 fallback 到對話回報。
8. 寄信失敗也不得繼續下一個 item。

寄信格式：

```text
From: <notify_email_from>
To: <notify_email_to>
Subject: [DDD Queue Blocked] <item id> <item title> 需要確認

已完成：
- <item id> <title>, commit: <hash>

目前阻塞：
- <item id> <title>

阻塞原因：
<blocker_reason>

需要你確認：
- A: ...
- B: ...

Queue 已暫停，尚未執行後續 item。
```

## Queue 完成通知

orchestrator 每完成一個 item 後，檢查 queue 是否全部完成。只有同時符合以下條件才算 queue completed：

1. 所有非 skipped item 都是 `status: completed`。
2. 沒有 `pending`、`in_progress` 或 `blocked` item。
3. 不是因達到 `batch_limit` 而暫停。
4. 每個 completed item 都有 `commit` 與 `implemented_doc`。

queue completed 後：

1. 停止 queue。
2. 依 `ddd-email-notify` 讀取 `notify_email_from`、`notify_email_to` 與 `notify_on_queue_completed`。
3. 若 `notify_on_queue_completed: true` 且寄信設定可用，寄出 `[DDD Queue Completed]` 通知。
4. 寄信成功或失敗都追加 `notification` ledger entry，記錄 From、To、subject、delivery status 與是否 fallback 到對話回報。
5. 若無可用寄信設定或工具，直接在目前對話回報 queue completed，並說明未寄信原因。

寄信格式：

```text
From: <notify_email_from>
To: <notify_email_to>
Subject: [DDD Queue Completed] <QXX title>

Queue 已全部完成：
- <item id> <title>, commit: <hash>

測試：
- <主要命令與結果>

Queue 文件：
<queue file path>
```

## 最終回報

orchestrator 停止時回報：

```markdown
DDD queue 執行結束。

已完成：
- Q01-01 <title> — <commit>
- Q01-02 <title> — <commit>

已阻塞：
- Q01-03 <title> — <原因>

未執行：
- Q01-04 <title>

測試：
- <各 item 執行過的主要命令>

溝通紀錄：
- Agent Communication Ledger: L001-L0XX

通知：
- ddd-email-notify: sent / skipped-not-configured / failed / not-completed
- From: <notify_email_from 或 —>
- To: <notify_email_to 或 —>
- Reason: <若未寄信，說明原因>

Queue 文件：
- documents/queue/Q01-xxx.md
```
