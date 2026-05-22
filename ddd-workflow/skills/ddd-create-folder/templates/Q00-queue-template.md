---
author: <作者>
date: <YYYY-MM-DD>
title: <一句話說明這批工作佇列>
uuid: d1f21e4ca9284a1b8b0c33f9cc2b9a5d
version: 1.0
status: pending
batch_limit: 3
mode: sequential
communication_format_version: 1
notify_email: <可選，阻塞時通知的 email>
---

# Queue - XXX 長時間工作佇列

## 1. 使用時機

本文件用於 2 到 5 個已排序、可自動推進的 DDD 工作。每個 item 會由新的 Codex 或 Claude Code session 單獨處理，完成後建立 git commit。

item 可以彼此獨立，也可以連續相依。若相依，必須填寫 `depends_on` 與 `unlock_condition`。

若後續 item 的範疇必須等前一 item 完成後才知道，或階段拆分本身需要人類審查，請改用 `documents/planning/PXX` 規劃書。

## 2. 總覽

| Item | 名稱 | 類型 | Agent | Depends On | 狀態 | 關聯文檔 | Commit |
|------|------|------|-------|------------|------|----------|--------|
| QXX-01 | <工作名稱> | FXX | codex | — | pending | — | — |
| QXX-02 | <工作名稱> | FXX | claude | QXX-01 | pending | — | — |
| QXX-03 | <工作名稱> | BXX | auto | QXX-02 | pending | — | — |

---

## 3. Agent Communication Ledger (Append-only)

> 本區是 Codex、Claude Code、orchestrator 與使用者的共享通訊帳本。只能追加，不要刪改既有 entry。若要修正，新增 `correction` entry。

### Log Index

| Entry | Time | Item | From -> To | Type | Summary |
|-------|------|------|------------|------|---------|
| L001 | <YYYY-MM-DD HH:MM> | QXX-01 | orchestrator -> codex | dispatch | <指派 QXX-01> |

### Entries

#### L001 — <YYYY-MM-DD HH:MM> — QXX-01 — orchestrator -> codex — dispatch

**Message**
<指派內容。例：請以 ddd-start -> ddd-doc -> ddd-tdd 處理 QXX-01，完成後更新 queue 並 commit。>

**Context**
- Depends on: []
- Unlock condition: 無
- Relevant docs: —

**Expected Response**
- 建立或更新 F/R/B 文檔
- 記錄紅燈與綠燈測試
- 更新 item 狀態、handoff_summary、commit hash

**Artifacts**
- —

**Follow-up**
- —

---

## QXX-01 <工作名稱>

id: QXX-01
type: FXX
agent: codex
status: pending
depends_on: []
unlock_condition: 無
auto_approve: true
commit_required: true
implemented_doc: —
commit: —
worker_session: —
worker_log: —
handoff_summary: —
communication_entries: [L001]

### 需求

<清楚描述要完成的使用者行為。>

### 驗收方式

- [ ] <使用者可操作的確認動作，以及預期看到什麼結果。>
- [ ] <另一個確認動作。>

### 停止條件

- <需求不清楚時應停止的條件。>
- <需要使用者決策時應停止的條件。>

### 阻塞記錄

尚無。

### 子 Session 溝通

questions: []
need_user_decision: []
ledger_entries: [L001]

### Agent Handoff Summary

- Current state: pending
- Important files: —
- Decisions: —
- Tests: —
- Risks: —
- Next agent notes: —

---

## QXX-02 <工作名稱>

id: QXX-02
type: FXX
agent: claude
status: pending
depends_on: [QXX-01]
unlock_condition: QXX-01 completed 且相關基礎行為已有測試通過
auto_approve: true
commit_required: true
implemented_doc: —
commit: —
worker_session: —
worker_log: —
handoff_summary: —
communication_entries: []

### 需求

<清楚描述要在 QXX-01 基礎上完成的下一個使用者行為。>

### 驗收方式

- [ ] <使用 QXX-01 完成後的行為作為前提，確認此 item 的預期結果。>

### 停止條件

- <QXX-01 未完成或 commit 缺失。>
- <解鎖條件無法從文件、測試或程式碼確認。>

### 阻塞記錄

尚無。

### 子 Session 溝通

questions: []
need_user_decision: []
ledger_entries: []

### Agent Handoff Summary

- Current state: pending
- Important files: —
- Decisions: —
- Tests: —
- Risks: —
- Next agent notes: —

---

## QXX-03 <工作名稱>

id: QXX-03
type: BXX
agent: auto
status: pending
depends_on: [QXX-02]
unlock_condition: QXX-02 completed 且使用者流程已可執行
auto_approve: true
commit_required: true
implemented_doc: —
commit: —
worker_session: —
worker_log: —
handoff_summary: —
communication_entries: []

### 需求

<清楚描述錯誤現象與預期行為，或在前兩個 item 後要補上的修正。>

### 驗收方式

- [ ] <重現原本問題，確認修正後不再發生。>

### 停止條件

- <無法重現問題。>
- <需要使用者提供資料、帳號或操作步驟。>

### 阻塞記錄

尚無。

### 子 Session 溝通

questions: []
need_user_decision: []
ledger_entries: []

### Agent Handoff Summary

- Current state: pending
- Important files: —
- Decisions: —
- Tests: —
- Risks: —
- Next agent notes: —

## 4. Queue 執行規則

1. 每個 item 必須由新的 Codex 或 Claude Code session 處理。
2. 每個 item 必須以 `ddd-start → ddd-doc → ddd-tdd` 完整流程處理。
3. 每個 item 完成後必須 git commit。
4. 有相依關係時，必須確認 `depends_on` 全部 completed 且 `unlock_condition` 可驗證，才能開始下一個 item。
5. 所有跨 agent 溝通必須追加到 `Agent Communication Ledger`，並將 entry id 寫回對應 item 的 `communication_entries` / `ledger_entries`。
6. 子 session 需要使用者回答時，必須寫入 `questions` / `need_user_decision`，追加 `question` 或 `blocked` ledger entry，並將 item 設為 blocked。
7. 使用者回答後，orchestrator 必須追加 `answer` ledger entry，再啟動新的 worker session。
8. 遇到 blocked 必須停止整個 queue，不得繼續後續 item。
9. 一次最多處理 `batch_limit` 個 pending item，且不得超過 5 個。
