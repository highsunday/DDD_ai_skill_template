---
author: <作者>
date: <YYYY-MM-DD>
title: <一句話說明這批工作佇列>
uuid: d1f21e4ca9284a1b8b0c33f9cc2b9a5d
version: 1.0
status: pending
batch_limit: 3
mode: sequential
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

### 需求

<清楚描述要在 QXX-01 基礎上完成的下一個使用者行為。>

### 驗收方式

- [ ] <使用 QXX-01 完成後的行為作為前提，確認此 item 的預期結果。>

### 停止條件

- <QXX-01 未完成或 commit 缺失。>
- <解鎖條件無法從文件、測試或程式碼確認。>

### 阻塞記錄

尚無。

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

### 需求

<清楚描述錯誤現象與預期行為，或在前兩個 item 後要補上的修正。>

### 驗收方式

- [ ] <重現原本問題，確認修正後不再發生。>

### 停止條件

- <無法重現問題。>
- <需要使用者提供資料、帳號或操作步驟。>

### 阻塞記錄

尚無。

## 3. Queue 執行規則

1. 每個 item 必須由新的 Codex 或 Claude Code session 處理。
2. 每個 item 完成後必須 git commit。
3. 有相依關係時，必須確認 `depends_on` 全部 completed 且 `unlock_condition` 可驗證，才能開始下一個 item。
4. 遇到 blocked 必須停止整個 queue，不得繼續後續 item。
5. 一次最多處理 `batch_limit` 個 pending item，且不得超過 5 個。
