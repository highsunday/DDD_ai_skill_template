---
name: ddd-create-folder
description: 在新專案中建立 DDD 工作流程所需的資料夾結構與文檔模板。建立 documents/ddd-email-notify.md、documents/implements/（含 F00 / R00 / B00 模板）、documents/planning/（含 P00 模板）、documents/queue/（含 Q00 長時間工作佇列模板與 logs/ 歸檔資料夾）、documents/modules/ 與 documents/guides/（含 G00 模板）。當在新專案中初始化 DDD 工作流程時使用。
---

# DDD Create Folder

在當前專案中建立 DDD 工作流程所需的完整資料夾結構與初始模板檔案。

## 步驟一 — 確認建立位置

在當前工作目錄（專案根目錄）建立以下結構：

```
documents/
├── ddd-email-notify.md ← 專案層寄信通知設定
├── implements/     ← FXX / RXX / BXX 工作文檔存放處
│   ├── F00-feature-template.md
│   ├── R00-refactor-template.md
│   └── B00-bugfix-template.md
├── planning/       ← PXX 多階段規劃書存放處
│   └── P00-planning-template.md
├── queue/          ← QXX 長時間工作佇列存放處
│   ├── Q00-queue-template.md
│   └── logs/       ← queue 長 log / archive 存放處
├── modules/        ← 模組高層次文檔存放處（初始為空）
└── guides/         ← GXX 操作指南存放處（執行測試、打包等）
    └── G00-guide-template.md
```

若 `documents/` 資料夾已存在，告知使用者並詢問是否繼續（避免覆蓋現有內容）。

## 步驟二 — 建立資料夾

執行以下指令：

```bash
mkdir -p documents/implements
mkdir -p documents/planning
mkdir -p documents/queue
mkdir -p documents/queue/logs
mkdir -p documents/modules
mkdir -p documents/guides
```

## 步驟三 — 寫入模板檔案

讀取本技能資料夾（`skills/ddd-create-folder/templates/`）中的模板，將其內容原封不動寫入專案：

- `templates/F00-feature-template.md` → `documents/implements/F00-feature-template.md`
- `templates/DDD-email-notify-template.md` → `documents/ddd-email-notify.md`
- `templates/R00-refactor-template.md`  → `documents/implements/R00-refactor-template.md`
- `templates/B00-bugfix-template.md`   → `documents/implements/B00-bugfix-template.md`
- `templates/P00-planning-template.md` → `documents/planning/P00-planning-template.md`
- `templates/Q00-queue-template.md`    → `documents/queue/Q00-queue-template.md`
- `templates/G00-guide-template.md`    → `documents/guides/G00-guide-template.md`

若目標檔案已存在，**不覆蓋**，並告知使用者跳過了哪些檔案。

## 步驟四 — 建立 CONTEXT.md（可選）

詢問使用者：「是否要在專案根目錄建立 `CONTEXT.md` 領域語言模板？」

若同意，在專案根目錄建立以下內容的 `CONTEXT.md`：

```markdown
# CONTEXT.md — 專案領域語言

> 這是專案的領域詞彙表。所有 AI 技能在撰寫文檔、測試或代碼之前都會讀取此檔案。
> 請填入你的專案詞彙，刪除不適用的範例。

## Language（術語定義）

**[術語 1]**：
[定義]
_避免使用：_ [近義詞]

## Relationships（關係說明）

## Architecture Boundaries（架構邊界）

## Flagged Ambiguities（已釐清的模糊點）
```

若根目錄已有 `CONTEXT.md`，不覆蓋，告知使用者已跳過。

## 步驟五 — 報告結果

完成後輸出建立摘要：

```
DDD 資料夾初始化完成：

已建立：
  ✓ documents/implements/
  ✓ documents/planning/
  ✓ documents/queue/
  ✓ documents/queue/logs/
  ✓ documents/modules/
  ✓ documents/guides/
  ✓ documents/ddd-email-notify.md
  ✓ documents/implements/F00-feature-template.md
  ✓ documents/implements/R00-refactor-template.md
  ✓ documents/implements/B00-bugfix-template.md
  ✓ documents/planning/P00-planning-template.md
  ✓ documents/queue/Q00-queue-template.md
  ✓ documents/guides/G00-guide-template.md
  [✓ CONTEXT.md（若使用者選擇建立）]

已跳過（已存在）：
  [列出跳過的檔案，若無則省略]

下一步：
  1. 填寫 CONTEXT.md 定義你的專案領域術語
  2. 填寫 documents/ddd-email-notify.md 的 notify_email_from / notify_email_to（若需要完成或 blocked 通知）
  3. 執行 /ddd-start 開始第一個工作
```
