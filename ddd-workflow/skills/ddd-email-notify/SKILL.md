---
name: ddd-email-notify
description: DDD workflow 的操作通知技能。用於顯示目前寄信設定、設定或寄出 ddd-tdd completed、queue blocked、queue completed 通知信，維護 project config 或 QXX frontmatter 的 notify_email_from 與 notify_email_to，驗證寄信來源是否為目前環境已授權帳號，並在寄信成功或失敗後寫入 Agent Communication Ledger。不要用於大量行銷信或一般非 DDD 通知。
---

# DDD Email Notify

處理 DDD workflow 的操作通知信。主要由 `ddd-tdd` 與 `ddd-queue` 呼叫，也可在使用者要求「設定通知信箱」或直接輸入 `/ddd-email-notify` 時使用。

## 直接觸發時的行為

當使用者直接輸入 `/ddd-email-notify`，且沒有要求寄信或改設定時，顯示目前寄信 info，不寄信：

```markdown
DDD Email Notify

設定來源：
- <documents/ddd-email-notify.md 或目前 QXX queue frontmatter；若都沒有，顯示未設定>

寄信設定：
- From: <notify_email_from 或 未設定>
- To: <notify_email_to 或 未設定>

寄信時機：
- `/ddd-tdd` 單獨完成：會寄信（若 notify_on_tdd_completed: true 且 From/To 可用）
- `/ddd-queue` blocked：會寄信（若 notify_on_queue_blocked: true 且 From/To 可用）
- `/ddd-queue` 全部完成：會寄信（若 notify_on_queue_completed: true 且 From/To 可用）
- `/ddd-queue` worker 內部呼叫的 `/ddd-tdd` 完成：不寄信，避免每個 item 重複通知

目前限制：
- <寄信工具是否可用；若無法確認，說明會在實際寄信前驗證>
```

## 設定欄位

專案層設定檔放在：

```text
documents/ddd-email-notify.md
```

project config 或 QXX frontmatter 保存以下欄位：

```yaml
notify_email_from: <寄信來源；必須是目前環境已授權可寄出的信箱>
notify_email_to: <寄去哪裡；通知收件信箱>
notify_on_tdd_completed: true
notify_on_queue_blocked: true
notify_on_queue_completed: true
notify_email: <deprecated；舊版收件信箱，遷移到 notify_email_to>
```

規則：

1. 不保存 email 密碼、token、SMTP key、app password。
2. `notify_email_from` 是寄件身分，不是 provider 名稱。
3. `notify_email_to` 是收件信箱。
4. 舊 QXX 若只有 `notify_email`，視為 `notify_email_to`，並在下次更新時補上 `notify_email_to`。
5. 如果使用者要求完成或 blocked 時寄信，必須同時有 `notify_email_from` 與 `notify_email_to`。
6. QXX frontmatter 優先於 project config；缺少的欄位可從 `documents/ddd-email-notify.md` 補齊。
7. `/ddd-queue` worker 內部呼叫 `/ddd-tdd` 時，必須抑制 `notify_on_tdd_completed`，避免每個 item 完成時寄信。

## 設定流程

當使用者要求設定寄信功能：

1. 若使用者指定 QXX，讀取 QXX frontmatter；否則讀取或建立 `documents/ddd-email-notify.md`。
2. 若缺少 `notify_email_from` 或 `notify_email_to`，請使用者提供：
   - 寄信來源：`notify_email_from`
   - 寄去哪裡：`notify_email_to`
3. 驗證兩者至少符合基本 email 格式：包含一個 `@`，且本地端與網域端都非空。
4. 設定寄信時機：
   - `notify_on_tdd_completed: true`
   - `notify_on_queue_blocked: true`
   - `notify_on_queue_completed: true`
   除非使用者明確關閉其中一項。
5. 更新 QXX frontmatter 或 `documents/ddd-email-notify.md`。
6. 不測試寄信，除非使用者明確要求。

## 寄信前檢查

寄信前必須確認：

1. `notify_email_to` 存在。
2. `notify_email_from` 存在。
3. 目前環境有可用寄信工具，例如 Gmail connector、專案提供的通知腳本、`sendmail` 或 `mail`。
4. 若工具只能用目前登入帳號寄信，必須確認該帳號與 `notify_email_from` 一致。
5. 若無法驗證寄信來源，不寄信，改在目前對話回報當前 blocked 或 completed 狀態。

## 寄信時機

### ddd-tdd standalone completed

`/ddd-tdd` 單獨完成一份 F/R/B 實作時寄信，條件：

1. 實作狀態是 `已實作`。
2. 紅燈、綠燈、驗收與文檔更新都完成。
3. 不是由 `/ddd-queue` worker 呼叫。
4. `notify_on_tdd_completed: true`。
5. `notify_email_from` 與 `notify_email_to` 可用且寄件來源通過驗證。

若 `/ddd-tdd` 是由 `/ddd-queue` worker 呼叫，永遠不寄單項完成通知；由 queue orchestrator 在整批全部完成時寄一次。

### ddd-queue blocked

`/ddd-queue` 有 item blocked 時寄信，條件：

1. queue 已停止。
2. `notify_on_queue_blocked: true`。
3. `notify_email_from` 與 `notify_email_to` 可用且寄件來源通過驗證。

### ddd-queue completed

`/ddd-queue` 全部完成時寄信，條件：

1. queue 內所有非 skipped item 都是 `completed`。
2. 沒有 pending / in_progress / blocked item。
3. 不是只達到 batch limit 後暫停。
4. `notify_on_queue_completed: true`。
5. `notify_email_from` 與 `notify_email_to` 可用且寄件來源通過驗證。

## 寄信內容

blocked 通知格式：

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

ddd-tdd completed 通知格式：

```text
From: <notify_email_from>
To: <notify_email_to>
Subject: [DDD TDD Completed] <F/R/B id> <title>

已完成：
- 文檔：<path>
- 實作狀態：已實作

測試：
- <command>: <result>

變更摘要：
<summary>
```

queue completed 通知格式：

```text
From: <notify_email_from>
To: <notify_email_to>
Subject: [DDD Queue Completed] <QXX title>

Queue 已全部完成：
- <QXX-01 title>, commit: <hash>
- <QXX-02 title>, commit: <hash>

測試：
- <主要命令與結果>

Queue 文件：
<path>
```

## Delivery 結果

每次嘗試通知後，回傳並寫入 `Agent Communication Ledger`：

```md
#### LXXX — <time> — <item> — orchestrator -> user — notification

**Message**
DDD queue blocked notification.

**Context**
- From: <notify_email_from>
- To: <notify_email_to>
- Subject: <subject>

**Artifacts**
- Delivery status: sent | draft-created | failed | fallback-dialog
- Tool: <工具名稱或 unavailable>

**Follow-up**
- <若失敗，說明使用者需要如何補設定>
```

若 queue 的寄信失敗或只能建立草稿，不得繼續下一個 queue item。standalone `/ddd-tdd` 寄信失敗時，在最終回報中說明通知失敗，但不要回滾已完成的實作。
