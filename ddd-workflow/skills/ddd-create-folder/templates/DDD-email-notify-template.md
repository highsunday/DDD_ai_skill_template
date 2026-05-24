---
notify_email_from: <可選，寄信來源；必須是目前環境已授權可寄出的信箱>
notify_email_to: <可選，通知收件信箱>
notify_on_tdd_completed: true
notify_on_queue_blocked: true
notify_on_queue_completed: true
---

# DDD Email Notification Settings

本文件保存 DDD workflow 的專案層通知設定。只保存信箱地址與寄信時機，不保存 email 密碼、token、SMTP key 或 app password。

## Current Settings

- From: `notify_email_from`
- To: `notify_email_to`
- Standalone ddd-tdd completed: `notify_on_tdd_completed`
- DDD queue blocked: `notify_on_queue_blocked`
- DDD queue completed: `notify_on_queue_completed`

## Notification Rules

- `/ddd-tdd` 單獨完成一份 F/R/B 實作時，寄完成通知。
- `/ddd-queue` 整批全部完成時，寄完成通知。
- `/ddd-queue` blocked 時，寄 blocked 通知。
- 由 `/ddd-queue` worker 呼叫的 `/ddd-tdd` 不寄單項完成通知；只由 `/ddd-queue` 在整批全部完成時寄一次。
- 若寄信工具不可用或寄件來源無法驗證，改在目前對話回報，不保存任何 credential。
