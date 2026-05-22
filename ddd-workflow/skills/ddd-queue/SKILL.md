---
name: ddd-queue
description: "DDD 長時間工作佇列技能。建立或執行 documents/queue/ 下的 QXX queue 文件，適用於 2 到 5 個已排序、可自動推進的功能、Bug 修正或重構工作；item 可以彼此獨立，也可以有明確 depends_on 與 unlock_condition。每個 item 必須由新的 Codex 或 Claude Code session 單獨處理，完成後 git commit，遇到 blocker 則寄信通知使用者並停止。不要用於單一工作、需求仍需人類逐階段決策、或後續階段範疇必須等前一階段完成後才能定義的工作，這些分別使用 ddd-doc/ddd-tdd 或 ddd-plan。"
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
depends_on: []
unlock_condition: 無
auto_approve: true
commit_required: true
implemented_doc: —
commit: —

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
- `depends_on`: 依賴的 item id 清單，例如 `[Q01-01]`；無依賴時填 `[]`
- `unlock_condition`: 何時可以開始此 item，例如 `Q01-01 completed 且登入頁已可正常提交`
- `auto_approve`: `true` 表示可由 worker 自行建立 F/R/B 文檔並視為已核准；但 worker 仍要在文檔清楚後才開始改程式碼。
- `implemented_doc`: 完成後填入實際文件路徑，例如 `documents/implements/F03-user-search.md`
- `commit`: 完成後填入 commit hash

## 建立 Queue

當使用者要求排一批工作、延長 AI 自主工作時間、或減少逐項打斷時：

1. 讀取 `CONTEXT.md`，使用既有領域語言。
2. 建立 `documents/queue/`，並確認下一個可用的 `QXX` 編號。
3. 將每個需求整理成一個 item；每個 item 都必須有明確驗收方式。
4. 若 item 之間有依賴，寫入 `depends_on` 與 `unlock_condition`，並依依賴順序排序。
5. 若後續 item 的需求要等前一 item 完成後才知道，停止並建議改用 `ddd-plan`。
6. 若 item 太模糊，將該 item 標成 `blocked` 或詢問使用者，不要把模糊需求放進 `pending`。
7. 在 queue frontmatter 填入：
   - `status: pending`
   - `batch_limit: 3`，除非使用者指定 1 到 5
   - `notify_email`，若使用者已提供

完成後回報 queue 文件路徑與 item 清單。若使用者同時要求執行，繼續「執行 Queue」。

## 執行 Queue

AI 目前所在 session 是 orchestrator。orchestrator 只負責挑選 item、啟動新的子 session、檢查結果、寄信與停止；不要在 orchestrator session 直接實作功能。

### 步驟 1 — 前置檢查

1. 讀取 queue 文件。
2. 確認 `git status --short` 為乾淨。若不乾淨，停止並請使用者先處理；不要自動 commit 使用者既有變更。
3. 確認 `codex` / `claude` CLI 是否存在：
   - Codex: `codex exec`
   - Claude Code: `claude -p`
4. 決定本輪最多處理幾個 item：使用使用者指定值，否則使用 queue 的 `batch_limit`，再否則預設 3；絕不可超過 5。

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
3. 開始前確認工作樹乾淨；若不乾淨，回報 blocked，不要 commit。
4. 檢查 depends_on；所有依賴 item 必須 completed 且有 commit，unlock_condition 必須可驗證。
5. 閱讀依賴 item 的 implemented_doc、commit 摘要與測試記錄，建立本 item 所需上下文。
6. 將指定 item 標成 in_progress。
7. 根據 item type 建立或更新對應 FXX / BXX / RXX 文檔。
8. 若 auto_approve: true 且需求、驗收方式、停止條件、依賴與解鎖條件都清楚，將該文檔視為本 item 的已核准文檔。
9. 依照 ddd-tdd 執行紅燈、綠燈、驗收與文檔更新。
10. 若需求模糊、高風險、測試無法穩定通過、需要使用者決策，將 item 標成 blocked，填寫 blocker_reason 與 need_user_decision，停止。
11. 完成時更新 queue item：status: completed、implemented_doc、commit。
12. git commit 必須只包含本 item 相關檔案；使用明確 git add 檔案清單，不要使用 git add .
13. commit 後確認工作樹乾淨。
14. 最終回覆只回報 item 狀態、commit hash、執行測試命令與任何限制。不要開始下一個 item。
```

## Worker 完成規則

worker 完成 item 前必須：

1. 建立或更新一份 F/R/B 文檔。
2. 依該文檔新增或更新測試。
3. 執行相關測試，必要時執行更廣的回歸測試。
4. 更新 F/R/B 文檔的實作記錄。
5. 更新 queue item 狀態與總覽表。
6. 建立 git commit。
7. 確認 `git status --short` 乾淨。

建議 commit message：

```text
feat: complete Q01-01 <功能名稱>
fix: complete Q01-02 <修正名稱>
refactor: complete Q01-03 <重構名稱>
```

## Blocker 規則

以下情況必須 blocked：

- 需求缺少可驗收行為。
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
need_user_decision:
- A: <選項>
- B: <選項>
```

不要 commit 不完整的生產代碼。若已產生部分變更且無法安全完成，留下清楚記錄並停止；orchestrator 不得繼續下一個 item。

## 通知使用者

orchestrator 偵測到 blocked 後：

1. 停止 queue。
2. 使用 queue frontmatter 的 `notify_email` 寄信；若沒有 email 或沒有可用寄信工具，直接在目前對話回報 blocked。
3. 寄信失敗也不得繼續下一個 item。

寄信格式：

```text
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

若整批正常完成，不必寄信，除非使用者要求完成通知。

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

Queue 文件：
- documents/queue/Q01-xxx.md
```
