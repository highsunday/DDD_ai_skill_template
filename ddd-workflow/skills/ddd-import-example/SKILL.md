---
name: ddd-import-example
description: 參考 reference-examples/import/ 下、由 ddd-export-example 從別專案導出的最小可跑範例，在目前專案實作類似功能。讀懂範例與 MANIFEST、對齊本專案語境（同棧為主、跨棧靠 MANIFEST 的核心概念儘力翻譯），生成一份引用該範例的 FXX 草稿、把範例 smoke test 翻成本專案的驗收條件，人類審核後交給 ddd-tdd 實作。當使用者已把別專案的範例 bundle 複製進 reference-examples/import/、想照它在本專案加上同類功能時使用。
---

# DDD Import Example

參考別專案導出的最小可跑範例，在目前專案實作類似功能。匯出端是 `ddd-export-example`；本技能不自己跨 repo 抓檔，範例由使用者**手動複製**到 `reference-examples/import/` 後才執行。

## 在 DDD pipeline 中的位置

這個技能**不直接改程式碼**。它把「外來範例」轉成「本專案的 FXX 文檔」，再交回既有 DDD 流程：

```
別專案 export bundle  →（使用者手動複製）→ reference-examples/import/<feature>/
        ↓ ddd-import-example
理解範例 + 對齊本專案語境
        ↓
生成引用範例的 FXX 草稿（含驗收條件 = 翻譯後的 smoke test）
        ↓ 人類審核
ddd-tdd 紅燈 → 綠燈 → 驗收
```

信條：**沒有模糊文檔進入 ddd-tdd。** 範例再好，也要先落成一份本專案語境的 FXX。

---

## 步驟一 — 定位範例 bundle

1. 列出 `reference-examples/import/` 下的 bundle。
2. 若資料夾不存在或為空：提醒使用者「請先用 ddd-export-example 在來源專案導出，再把該 bundle 資料夾複製到本專案的 `reference-examples/import/`」，然後停止。
3. 若有多個，詢問使用者這次要參考哪一個。

---

## 步驟二 — 讀懂範例

依序讀：

1. **`MANIFEST.md`**（先讀）——功能摘要、入口、檔案地圖、核心概念、改寫指引、smoke test 摘要。
2. **入口檔 → 依檔案地圖讀核心程式碼**——理解本質流程。
3. **smoke test**——理解「能跑」具體驗證了什麼。

向使用者用幾句話複述「我理解這個範例在做什麼」，確認沒理解錯再往下。

---

## 步驟三 — 對齊本專案語境

讀本專案資訊以對齊：

- `CONTEXT.md`（若存在）——術語、架構邊界。
- `documents/modules/` 中相關模組文檔——既有職責邊界。
- 本專案的依賴清單 / 既有同類程式碼——確認技術棧與慣例。

判斷同棧或跨棧：

- **同棧（語言 / 框架相同）**：以範例的**具體程式碼**為主要參考，逐段對應到本專案結構與命名。
- **跨棧（語言不同）**：以 MANIFEST 的**「核心概念（語言無關）」段**為主要參考，依本質流程在本專案的棧上**儘力翻譯**重建；具體程式碼僅作輔助。

把「哪些保留、哪些要替換」對照 MANIFEST 的「改寫指引」整理出來：

```
對齊計畫（功能：<feature>，同棧 / 跨棧）：

保留（功能本質）：
  - <核心邏輯>

替換成本專案語境：
  - <範例的 fake DB> → 本專案的 <真實資料層>
  - <範例命名> → 本專案 CONTEXT.md 對應術語

需要使用者裁示：
  - <模糊或缺資訊的點>
```

---

## 步驟四 — 生成 FXX 草稿

1. 在 `documents/implements/` 找下一個可用編號（F01、F02…；目錄不存在則 `mkdir -p documents/implements`）。
2. 依本專案的 F00 模板（`documents/implements/F00-feature-template.md`，若有）撰寫 FXX，重點：
   - **功能概述**：說明本功能，並註明「參考範例：`reference-examples/import/<feature>/`」。
   - **需求描述**：以本專案使用者角色撰寫 User Story。
   - **驗收準則 / 測試情境**：把範例 MANIFEST 的 **smoke test 摘要翻譯成本專案語境的 Given-When-Then**，作為至少一條驗收條件。範例「能跑」的那條路徑，就是本專案要先跑綠燈的那條。
   - **實作註記**：附對齊計畫——哪些照搬、哪些替換、跨棧時的翻譯重點，並標注 MANIFEST 中的「容易踩雷的點」。
3. 寫入 `documents/implements/F0X-<feature>.md`。

---

## 步驟五 — 交回人類審核與 ddd-tdd

回報並停在審核點：

```
FXX 草稿已生成（參考範例：reference-examples/import/<feature>/）：

  ✓ documents/implements/F0X-<feature>.md

對齊方式：<同棧，直接參考程式碼 / 跨棧，依核心概念翻譯>
驗收條件來源：範例 smoke test → 已翻成本專案 Given-When-Then

下一步：
  1. 請審核這份 FXX（特別是驗收條件與「替換」清單是否正確）。
  2. 審核通過後執行 /ddd-tdd，由它驅動紅燈 → 綠燈 → 驗收實作。
```

**不要自己跳過審核直接實作。** 審核通過後由 `ddd-tdd` 接手。
