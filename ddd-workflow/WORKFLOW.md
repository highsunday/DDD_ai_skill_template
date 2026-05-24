# AI 技能模板 — 強化版方法摘要（DDD × Pocock）

本專案在原有 DDD 工作流程的基礎上，整合 Matt Pocock 的技能庫，針對四個失敗節點進行強化。核心理念不變：文檔是唯一真理來源，AI 起草，人類審查，再由 AI 依此實作。新增的技能以**外掛點**的方式插入循環，而非替換骨架。

---

## 統一開發循環

```
人類提出需求
        ↓
[ddd-start]  偵測需求類型，路由到正確入口
        ↓
  ┌─ 模糊想法 ──→ [grill-me]        深挖真實需求
  ├─ 涉及架構 ──→ [grill-with-docs]  對照現有領域模型驗證
  ├─ 多階段大型改動 ──────────────────────────────────────────────────────┐
  │                                                                      ↓
  │                                             [ddd-plan]  草擬 PXX 規劃書
  │                                                                      │
  │                                                         人類審查並核准 PXX
  │                                                                      │
  │                                          逐階段執行（每個階段走以下流程）
  │                                                                      │
  ├─ 長時間工作佇列 ─────────────────────────────────────────────────────┐
  │                                                                      ↓
  │                                             [ddd-queue]  建立 QXX 佇列
  │                                                                      │
  │                            集中 [grill-me] 釐清所有 item
  │                                                                      │
  │                     每個 item 開新 session 跑 ddd-start/doc/tdd
  │                                                                      │
  └─ 已有規格 ──→ 直接進入文檔步驟 ←──────────────────────────────────────┘
        ↓
[ddd-doc]  草擬 FXX / RXX / BXX 文檔（若來自 PXX，依規劃書建議類型起草）
        │
        ├─ RXX 文檔時 → [zoom-out]  確認重構範疇邊界
        └─ 非正式描述 → [to-prd]    轉換為結構化草稿
        ↓
人類審查並核准文檔
        ↓
[ddd-tdd]  紅燈 → 綠燈 → 驗收
        │
        └─ 測試意外失敗 → [diagnose]  結構化除錯 → 結果寫回 FXX
        ↓
[ddd-doc]  同步 FXX 記錄 + 模組文檔
        │     若來自 PXX：回填關聯文檔編號、更新階段狀態
        │
        └─ 發現架構技術債 → [improve-codebase-architecture]  輸出 RXX 候選
```

這個循環將**人類置於規劃者與審查者的角色**。Pocock 技能在每個失敗節點提供結構化的恢復路徑，而非事後補救。

---

## 三大文檔類別

### 類別零 — 規劃書（`documents/planning/`）

**目的：** 將無法一次完成的大型改動拆成多個有序階段，作為後續 F/R/B 文件的上層規劃依據。

這些文檔在大型工作*開始前*由 AI 起草，在每個階段完成後*持續更新*狀態。

| 類型 | 前綴 | 模板 | 使用時機 |
|---|---|---|---|
| 多階段規劃 | `PXX` | `P00-planning-template.md` | 需要多個 F/R/B 才能完成的大型改動 |

每份 PXX 文檔包含：
- 背景、總體目標與影響範圍
- 各階段的描述與**使用者可驗證的預期成果**（讓使用者能自行確認每個階段是否成功）
- 建議的 F/R/B 文檔類型，及執行後回填的實際關聯文檔編號
- 各階段狀態（未開始 / 進行中 / 已完成）
- 接棒 AI 的執行指引，說明如何讀取本文件並推進下一個階段

---

### 類別零之一 — Queue（`documents/queue/`）

**目的：** 將多個已排序、可自動推進的工作排成佇列，讓 AI 逐一以 DDD 流程完成，且每個 item 使用新的 Codex / Claude Code session 與獨立 commit。

這些文檔適用於「我有 2 到 5 個工作想讓 AI 先連續做一段時間」的情境。item 可以獨立，也可以相依；相依時必須明確寫出 `depends_on` 與 `unlock_condition`。

若後續 item 的範疇必須等前一階段完成後才知道，或階段拆分本身需要人類審查，使用 `ddd-plan`。

| 類型 | 前綴 | 模板 | 使用時機 |
|---|---|---|
| 工作佇列 | `QXX` | `Q00-queue-template.md` | 多個已排序、可自動推進的功能、Bug 修正或重構 |

每份 QXX 文檔包含：
- batch limit 與 blocked / completed 通知設定；若需要寄信，使用 `documents/ddd-email-notify.md` 或 QXX frontmatter 明確設定 `notify_email_from`（寄信來源）與 `notify_email_to`（寄去哪裡）
- `Queue Intake Review`：在執行前集中使用 `grill-me` 釐清所有 item 的需求、設計問題、依賴、驗收方式與停止條件
- 每個 item 的類型（FXX/RXX/BXX）、指定 agent、依賴、解鎖條件、狀態、關聯文檔與 commit hash
- 每個 item 的需求、使用者可操作驗收方式與停止條件
- `Agent Communication Ledger`：append-only 記錄 orchestrator、Codex、Claude Code 與使用者之間的派工、問題、回答、決策、測試證據與接棒摘要；QXX 主文件只保留索引、摘要與 active entries，長 log 歸檔到 `documents/queue/logs/`
- blocked 時的原因與需要使用者決策的選項

執行時由 orchestrator 讀取 QXX，先確認 `Queue Intake Review` 已完成、`ready_for_execution: true`，且每個要執行的 item 都是 `clarification_status: clarified`。接著對每個 pending item 啟動新的 worker session。worker 只處理單一 item，且必須完整走 `ddd-start → ddd-doc → ddd-tdd`。完成後測試、更新文檔、git commit，然後停止。

非互動子 session 不做即時對話。需要使用者判斷時，worker 將 item 設為 blocked，寫入 `questions` / `need_user_decision`，並追加 ledger entry；orchestrator 通知使用者並停止 queue。若已設定 `notify_email_from` 與 `notify_email_to` 且目前環境有可用寄信工具，orchestrator 會寄出 blocked 通知；若缺少設定、寄件來源無法驗證或寄信失敗，則在目前對話回報 blocked，且不得繼續後續 item。queue worker 內部呼叫的 `/ddd-tdd` 不寄單項完成信；只有整批 queue 全部完成時，由 orchestrator 寄一次 completed 通知。使用者回答後，也由 orchestrator 追加 `answer` entry，讓後續 agent 能從 QXX 讀到必要溝通紀錄。為了控制 token，後續 worker 預設只讀 Queue Intake Review、指定 item、依賴 item handoff、Log Index 與相關 active entries；除非 blocked 或上下文矛盾，不讀完整 archive。

---

### 類別一 — 實作文檔（`documents/implements/`）

**目的：** 記錄並釐清每個交付給 AI 實作的工作單元範疇。

這些文檔在編碼*開始前*創建，在實作*完成後*更新。

| 類型 | 前綴 | 模板 | 使用時機 |
|---|---|---|---|
| 功能規格 | `FXX` | `F00-功能需求書模板.md` | 新功能 |
| 重構規格 | `RXX` | `R00-重構任務模板.md` | 不改變行為的結構性改善 |
| 錯誤修正規格 | `BXX` | `B00-Bug修正模板.md` | 重現並修正缺陷 |

每份文檔包含：
- 用戶故事與功能背景
- Given/When/Then 格式的驗收標準
- 測試場景表（ID、Given、When、Then、優先級）
- 實作說明與限制條件
- 編碼後由 AI 填入的**實作記錄**章節（狀態、測試證據、變更檔案、假設、缺口）

---

### 類別二 — 模組文檔（`documents/modules/`）

**目的：** 在高層次反映當前程式碼庫的狀態，讓新工程師無需閱讀每個檔案即可快速理解系統。

關鍵特性：
- **僅呈現高層次內容。** 職責、邊界、數據流、架構——而非逐行代碼細節。
- **始終與代碼同步。** 代碼與模組文檔不一致時，以代碼為準。
- **面向工程師。** 讓新成員能快速定位模組的作用、位置、依賴與已知限制。
- **不描述未實作的行為。**

---

## 六個 DDD 技能

### `ddd-start`（新增）

**用途：** 統一入口，偵測需求類型並路由到正確的下游技能，確保沒有模糊需求直接進入文檔撰寫。

**觸發時機：** 任何新工作開始時。

**行為：**
1. 讀取專案 `CONTEXT.md`（如存在），載入領域語言
2. 偵測需求類型：
   - **模糊想法**（缺乏具體行為描述）→ 執行 `grill-me`，收斂後交棒 `ddd-doc`
   - **涉及現有架構的新功能**（修改現有模組）→ 執行 `grill-with-docs`，交棒 `ddd-doc (FXX)`
   - **規格清晰的新功能** → 直接交棒 `ddd-doc (FXX)`
   - **重構任務** → 執行 `zoom-out` 確認範疇，交棒 `ddd-doc (RXX)`
   - **Bug 回報** → 直接交棒 `ddd-doc (BXX)`
   - **長時間工作佇列**（2-5 個已排序且可自動推進的 item）→ 交棒 `ddd-queue`
   - **多階段大型改動**（後續範疇依賴前期結果、需先規劃或人類審查階段切分）→ 交棒 `ddd-plan`
3. 交棒時傳遞：需求類型、已挖掘的上下文、建議的文檔前綴與編號

---

### `ddd-plan`（新增）

**用途：** 大型改動的規劃入口。將無法一次完成的需求分解為有序的執行階段，為每個階段定義使用者可實際操作確認的成果，起草 PXX 規劃書後由人類審查核准，再逐階段交棒 `ddd-doc` 執行。

**觸發時機：** 需求橫跨多個模組、階段切分需要人類審查、或後續階段範疇必須等前一階段完成後才能定義時。若相依 item 已能事先清楚列出並可自動推進，使用 `ddd-queue`。

**行為：**
1. 讀取 `CONTEXT.md`，載入領域語言
2. 若需求模糊，先執行 `grill-me` 深挖，再回來規劃
3. 盤點影響範圍，識別模組依賴與改動類型組合
4. 按「每個階段獨立可交付、對應一份 F/R/B 文檔、有可觀察成果」原則切分階段
5. 為每個階段設計**使用者可操作的確認方式**（不是技術指標）
6. 起草 PXX 規劃書，標注每個階段的建議文檔類型（FXX / RXX / BXX）
7. 人類審查核准後，PXX 成為後續所有 F/R/B 文檔的上層依據

**逐階段執行模式（PXX 核准後）：**

每個階段由使用者將 PXX 交給 AI，AI 讀取當前狀態並推進：
1. 讀取 PXX，找到最靠前的未開始階段
2. 呼叫 `ddd-doc` 起草對應的 FXX / RXX / BXX（一次一份）
3. 人類審查核准 F/R/B 後，呼叫 `ddd-tdd` 實作
4. 驗收通過後，回填 PXX 的關聯文檔編號並更新階段狀態
5. 重複至所有階段完成，將 PXX `status` 改為 `completed`

---

### `ddd-queue`（新增）

**用途：** 長時間工作佇列入口。建立或執行 QXX queue 文件，讓 2 到 5 個已排序、可自動推進的功能、Bug 修正或重構逐一完成。

**觸發時機：** 使用者列出多個工作，並希望 AI 一個接一個完成、每個功能新 session、每個功能完成後 git commit，或提到 queue / 佇列 / 批次自動執行 / 減少人被打斷。

**行為：**
1. 讀取 `CONTEXT.md`，使用專案領域語言整理 item
2. 建立 `documents/queue/QXX-*.md`，或讀取既有 QXX
3. 先用集中式 `grill-me` 釐清所有 item 的需求、設計問題、依賴、驗收方式與停止條件
4. 更新 `Queue Intake Review`，將可執行 item 標為 `clarification_status: clarified`
5. 驗證每個 item 都有明確需求、驗收方式與停止條件
6. 若 item 之間有依賴，寫入 `depends_on` 與 `unlock_condition`
7. 若後續 item 的範疇必須等前一階段完成後才知道，改用 `ddd-plan`
8. 執行時由 orchestrator 啟動新的 Codex / Claude Code worker session
9. worker 只處理指定 item，依序執行 `ddd-start`、`ddd-doc`、`ddd-tdd`、更新 queue、git commit
10. 所有跨 agent 訊息追加到 `Agent Communication Ledger`；長內容摘要留在 QXX，全文歸檔到 `documents/queue/logs/`
11. 完成 batch limit 或遇到 blocked 後停止
12. blocked 時使用 `notify_email_from` / `notify_email_to` 寄信通知使用者；若無可用寄信設定或工具，改在目前對話回報，並保留後續 item 為 pending
13. 整批 queue 全部完成時寄 completed 通知；達到 batch limit 但仍有 pending item 時不寄完成通知

---

### `ddd-email-notify`（新增）

**用途：** DDD workflow 的操作通知技能。負責顯示目前寄信 info，設定專案或 QXX 的寄信來源與收件信箱，並在 ddd-tdd completed、queue blocked、queue completed 時處理寄信與 ledger 記錄。

**觸發時機：** 使用者直接輸入 `/ddd-email-notify` 查看目前寄信 info，要求設定通知信箱，或 `ddd-tdd` / `ddd-queue` 需要寄信通知使用者。

**行為：**
1. 讀取 `documents/ddd-email-notify.md` 或 QXX frontmatter 的 `notify_email_from` 與 `notify_email_to`
2. 直接觸發時顯示目前 From、To、寄信工具狀態，以及 ddd-tdd completed / queue blocked / queue completed 的寄信時機
3. 若舊文件只有 `notify_email`，視為 `notify_email_to` 並在下次更新時遷移
4. 驗證兩個信箱欄位存在，且不將密碼、token、SMTP key 或 app password 寫入 QXX
5. 確認目前環境有可用寄信工具，並確認寄件來源與已授權帳號一致
6. 寄出 completed 或 blocked 通知；若無法寄信，改在目前對話回報
7. 將寄信成功、失敗或 fallback 結果追加到 `Agent Communication Ledger`

---

### `ddd-doc`（強化版 doc-maintain）

**用途：** 創建和維護兩類文檔。在原有 `doc-maintain` 基礎上新增三個外掛點。

**在開發循環中——編碼前：**

1. 讀取 `CONTEXT.md`，確保文檔術語與領域語言一致
2. 評估需求清晰度——若模糊，建議先執行 `grill-me` 或 `grill-with-docs`
3. 若描述非正式，可用 `to-prd` 先轉為結構化草稿，再填入 FXX 模板
4. 找到下一個可用的 FXX/RXX/BXX 編號並讀取對應模板
5. 檢查 `documents/modules/` 和程式碼庫以獲取相關上下文
6. **RXX 文檔專屬：** 鎖定範疇前執行 `zoom-out`，確認重構邊界不過窄也不過寬
7. 草擬完整的實作文檔供人類審查

**在開發循環中——編碼後：**

1. 以最終結果更新 FXX/RXX/BXX 實作記錄
2. 對照新實作審計受影響的模組文檔
3. 更新或標記已過時的模組文檔
4. 詢問：「實作過程中是否發現架構技術債？」若有，建議執行 `improve-codebase-architecture`，其輸出可作為新 RXX 文檔的候選草稿

---

### `ddd-tdd`（強化版 tdd-development）

**用途：** 所有生產代碼變更，由已核准的實作文檔驅動。在原有 `tdd-development` 基礎上新增三個外掛點。

**流程（不可協商）：**

1. 讀取 `CONTEXT.md`，確保測試命名與行為描述使用正確的領域詞彙
2. 讀取並理解 FXX/RXX/BXX 文檔——這是需求的唯一權威來源
3. 從文檔中提取驗收標準和測試案例
4. 撰寫最小的失敗測試（紅燈階段）——必須在繼續之前確認失敗
5. **若測試意外失敗（非「功能尚未實作」的正常紅燈）：** 進入 `diagnose` 結構化除錯迴圈，並將假設、根因、修復策略寫回 FXX 文檔的「實作記錄」
6. 實作通過測試所需的最小生產代碼（綠燈階段）
7. 驗證每個驗收標準都有測試覆蓋
8. 將實作記錄寫回文檔
9. 若本次不是 queue worker，依 `ddd-email-notify` 寄出 `/ddd-tdd` completed 通知；若是 queue worker，抑制單項完成信
10. 詢問：「實作過程中是否暴露架構缺口？」若有，建議 `improve-codebase-architecture`
11. 若為長時間實作週期，建議 `handoff` 保留 DDD 狀態

不從模糊的聊天指令開始。如果不存在實作文檔，停止並要求提供一份。

---

## 按需使用的 Pocock 技能（非強制步驟）

這些技能可以在 DDD 循環的任意節點手動觸發，不屬於強制流程：

| 技能 | 角色 | 典型觸發時機 |
|---|---|---|
| `caveman` | 壓縮溝通，減少 token 消耗 | 文檔審查過程中反覆確認措辭時 |
| `prototype` | 驗證方法的一次性探索 | 高度不確定的功能，提交 FXX 前 |
| `handoff` | 保留跨上下文重置的 DDD 狀態 | 長時間的實作週期 |
| `git-guardrails-claude-code` | 阻止破壞性的 git 命令 | tdd-tdd 期間的安全基礎設施 |

---

## 關鍵設計原則

- **AI 起草，人類核准。** 文檔創建步驟是協作的檢查點，而非形式。
- **沒有已核准的文檔，就沒有代碼。** `ddd-tdd` 將文檔視為其唯一提示。
- **沒有測試證據，就沒有成功。** 功能未完成，除非測試已運行並通過。
- **大型改動先規劃，再執行。** `ddd-plan` 確保多階段工作有清晰的地圖，每個階段都有人類可操作的驗收動作，而非模糊的「完成」狀態。
- **可自動推進的工作用 queue。** `ddd-queue` 適合 2 到 5 個已排序 item；可獨立也可相依，但依賴必須明確可驗證。每個 item 新 session、完成後獨立 commit、blocked 就停。
- **每次只執行一個階段。** PXX 核准後，逐階段起草 F/R/B 並實作，避免早期假設污染後期設計。
- **除錯是結構化的，而非臨時的。** 每次意外的測試失敗都經過 `diagnose` 處理，結果寫入 FXX 記錄。
- **架構觀察不丟失。** 實作後浮現的技術債通過 `improve-codebase-architecture` 轉化為 RXX 候選文檔。
- **模組文檔反映現實。** 過時的模組文檔被 `ddd-doc` 視為缺陷處理。
- **Pocock 技能是外掛點，不是替換。** `ddd-doc` 和 `ddd-tdd` 仍是核心執行引擎，Pocock 技能在失敗節點提供結構化恢復路徑。
