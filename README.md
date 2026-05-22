# 目標：DDD × Pocock 技能 — 統一 AI 開發工作流程

## 問題所在

我們的 DDD 工作流程（doc-maintain + tdd-development）解決了*結構*問題：它強制所有工作通過已審核的文檔，並執行 TDD 紀律。但仍存在以下缺口：

- **模糊的文檔產生模糊的代碼。** 沒有任何機制阻止一份寫得不好的 FXX 進入流水線。
- **TDD 之外的除錯是臨時性的。** 當測試意外失敗時，沒有結構化的恢復路徑。
- **跨會話的上下文會退化。** 長時間的實作週期會失去架構意識。
- **溝通過於冗長。** 文檔審查過程中的反覆確認消耗了大量 token，卻沒有提升品質。

Matt Pocock 的技能庫（[github.com/mattpocock/skills](https://github.com/mattpocock/skills)）解決了*品質*問題：它針對四種 AI 失敗模式——需求偏差、表達冗長、代碼無法運行、架構退化。他的技能可組合使用，但缺乏結構化的文檔系統作為錨點。

**目標：在每個失敗模式發生的節點，將 Pocock 的技能接入我們的 DDD 循環。**

---

## 統一循環

```
人類提出需求
        ↓
[grill-me / grill-with-docs]  ← Pocock：在撰寫文檔前挖掘真實需求
        ↓
[doc-maintain] 草擬 FXX / RXX / BXX          ← DDD：結構化文檔
        ↓
[zoom-out]（用於 RXX / 架構性變更）           ← Pocock：對照系統驗證範疇
        ↓
人類審查並核准文檔
        ↓
[tdd-development] 紅燈 → 綠燈 → 驗證         ← DDD：文檔驅動的 TDD
        ↑
[diagnose]（當測試意外失敗時）               ← Pocock：結構化除錯循環
        ↓
[doc-maintain] 同步 FXX 記錄 + 模組文檔       ← DDD：閉環
        ↓
[improve-codebase-architecture]（可選）       ← Pocock：浮現 RXX 候選項
```

多個已排序、可自動推進的工作可以改用 `ddd-queue`：先建立 `documents/queue/QXX`，在建立階段集中使用 `grill-me` 一次性釐清所有 item 的需求、設計問題、依賴、驗收與停止條件；QXX 完整更新且 ready 後，再由 orchestrator 每次開一個新的 Codex / Claude Code session 處理單一 item。每完成一個 item 就測試、更新文檔、git commit，做滿 batch limit 或遇到 blocker 後停止。item 可以獨立，也可以透過 `depends_on` 與 `unlock_condition` 明確相依。QXX 也會作為跨 agent 通訊帳本，但主文件只保留索引、摘要、未解問題與 handoff；長 stdout、完整對話與舊歷史歸檔到 `documents/queue/logs/`，避免每個 worker 重讀完整歷史。

---

## 技能對應表

### 第一階段 — 文檔撰寫之前

| Pocock 技能 | 在 DDD 循環中的角色 | 觸發時機 |
|---|---|---|
| `grill-me` | 在 `doc-maintain` 草擬任何內容之前，深入挖掘人類的需求 | 使用者只提供了模糊的想法 |
| `grill-with-docs` | 對照現有模組文檔和程式碼庫驗證需求 | 需求涉及現有架構 |
| `to-prd` | 將非正式描述轉換為結構化草稿，供 `doc-maintain` 填充 | 使用者有粗略想法但缺乏清晰規格 |

**為何重要：** FXX 文檔是 AI 在實作時的唯一提示。如果文檔薄弱，所有下游步驟都會退化。在撰寫前進行挖掘，是整個流水線中槓桿最高的介入點。

---

### 第二階段 — 文檔創建過程中

| Pocock 技能 / 概念 | 在 DDD 循環中的角色 | 備註 |
|---|---|---|
| `CONTEXT.md` 慣例 | `doc-maintain` 在草擬前讀取的共享領域語言 | 減少 FXX 文檔中的命名錯誤 |
| `caveman` | 文檔審查過程中的壓縮溝通 | 當人類與 AI 在措辭而非實質內容上反覆循環時使用 |
| `zoom-out` | RXX 範疇鎖定前的架構健全性檢查 | 防止 RXX 過窄或過寬 |

---

### 第三階段 — 實作過程中

| Pocock 技能 | 在 DDD 循環中的角色 | 觸發時機 |
|---|---|---|
| `diagnose` | 當測試因意外原因失敗時的結構化除錯循環 | 紅燈階段產生令人困惑或不穩定的失敗 |
| `triage` | BXX 錯誤修正文檔的狀態機工作流程 | 錯誤的重現路徑不清晰 |
| `prototype` | 在提交 FXX 文檔前驗證方法的一次性探索 | 高度不確定的功能 |
| `ddd-queue` | 多個已排序工作連續執行，先集中 grill-me 釐清所有 item，再逐項新 session 執行並各自 commit；QXX 保留精簡 ledger，長 log 歸檔 | 使用者希望 AI 連續完成 2 到 5 個可自動推進的功能或修正，減少人被打斷，且事後能回看 agent 溝通 |

---

### 第四階段 — 實作完成後

| Pocock 技能 | 在 DDD 循環中的角色 | 觸發時機 |
|---|---|---|
| `improve-codebase-architecture` | 浮現實作過程中發現的架構技術債 → 輸入新的 RXX 文檔 | 任何重要的綠燈階段完成後 |
| `handoff` | 保留跨上下文重置的 DDD 狀態的緊湊會話摘要 | 長時間的實作週期 |

---

### 基礎設施（持續啟用）

| Pocock 技能 | 角色 |
|---|---|
| `git-guardrails` | 在 tdd-development 期間阻止破壞性的 git 命令 |
| `write-a-skill` | 隨工作流程演進擴展技能庫的元技能 |

---

## 需要建構的內容

### 1. 讓 Pocock 的技能理解 DDD 上下文

Pocock 的技能針對通用程式碼庫運作。我們需要能夠感知以下內容的版本：
- `documents/implements/` — FXX/RXX/BXX 文檔存儲
- `documents/modules/` — 動態模組登記庫
- DDD 循環階段（文檔前 / 實作中 / 實作後）

具體而言：
- `grill-me` → 調整為輸出 FXX 草稿骨架，而非單純聊天
- `diagnose` → 解決後，將發現寫入 FXX 實作記錄
- `improve-codebase-architecture` → 輸出為 RXX 候選文檔，而非單純的文字描述

### 2. 加入 CONTEXT.md 慣例

在專案根目錄創建 `CONTEXT.md`（Pocock 的模式），定義：
- FXX 文檔中使用的領域詞彙
- 架構邊界
- 測試慣例
- 關鍵模組名稱與負責人

`doc-maintain` 和 `tdd-development` 在運作前都應讀取此文件。

### 3. 定義入口點路由

一個新的輕量技能（`ddd-start`）：
1. 接收人類的原始請求
2. 偵測其類型：新功能 / 重構 / 錯誤 / 模糊想法 / 長時間工作佇列
3. 路由到正確的入口點（grill-me → doc-maintain，ddd-queue → QXX，或 triage → BXX 等）
4. 攜帶正確上下文進行交接

---

## 成功標準

當以下條件成立時，工作流程即為成功：

1. **沒有模糊的 FXX 文檔進入 tdd-development。** 挖掘與 PRD 轉換在源頭攔截模糊性。
2. **除錯失敗是結構化的，而非臨時的。** 每次意外的測試失敗都經過 `diagnose` 處理，結果寫入 FXX 記錄。
3. **模組文檔無需手動維護即可保持最新。** `doc-maintain` + `improve-codebase-architecture` 在實作完成後立即標記架構漂移。
4. **多個工作可以連續執行但不失控。** `ddd-queue` 保證每個 item 有新 session、獨立 commit、明確依賴檢查、blocked 停止與通知。
5. **技能是可組合的，而非整體性的。** 每個 Pocock 技能都可以獨立使用或作為 DDD 循環的一部分——整合是增量的，而非重寫。
6. **新工程師僅憑文檔即可完成入職。** `CONTEXT.md` + 模組文檔 + FXX 歷史記錄完整呈現全貌，無需翻閱程式碼庫。

---

## 這不是什麼

- 不是替換現有的 DDD 技能——`doc-maintain` 和 `tdd-development` 仍然是核心執行引擎。
- 不是全面採用 Pocock 的系統——我們只取用能修補我們特定缺口的技能，而非整套哲學。
- 不是嚴格的循序流程——`zoom-out`、`caveman`、`handoff` 等技能是按需使用的工具，而非強制步驟。
