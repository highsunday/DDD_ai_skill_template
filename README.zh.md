# DDD × Pocock — AI 輔助開發工作流程

**DDD（Document-Driven Development）** 就是：先寫文檔，再寫代碼。

**AI 起草文檔 → 你審查核准 → AI 依文檔實作**

「文檔」不是事後補寫的說明，而是開始寫代碼前就要存在的需求契約。AI 不從聊天訊息直接寫代碼，只從你核准過的文檔寫代碼。

---

## 為什麼需要這套流程

沒有結構的 AI 開發，通常會遇到這四個問題：

1. **需求飄移** — 你說「做個登入」，AI 做完你才發現跟想的不一樣，但你也說不清楚哪裡不對，因為從頭到尾都沒有寫下來。
2. **不知道「完成」是什麼** — AI 說做完了，但你沒有標準確認。
3. **測試失敗就亂改** — AI 猜測性地改代碼，越改越亂，原因沒找到。
4. **技術債消失在對話裡** — 發現設計問題，但沒有地方記，下次再遇到重頭討論。

DDD 在流程的每個節點都有對應設計來攔截這四個問題。

---

## 三個不能打破的規則

| 規則 | 意思 |
|---|---|
| 沒有核准的文檔，不能寫代碼 | `ddd-tdd` 看不到文檔就停下來，強迫你先想清楚 |
| 沒有測試通過，不算完成 | 「應該沒問題」不算，要有測試證據才結案 |
| 測試意外失敗，要結構化除錯 | 走 `diagnose` 找根因，不是猜測性地亂改 |

---

## 角色分工

**你：** 提需求、審查文檔、核准、確認驗收結果
**AI：** 挖掘需求、起草文檔、寫代碼、更新記錄

核心只有一個：**AI 起草，但你核准**。核准這個動作是整個流程的品質閘門。

---

## 核心流程

```
你提出需求
    ↓
/ddd-start   判斷需求類型，必要時先挖掘清楚
    ↓
/ddd-doc     AI 起草文檔（FXX / RXX / BXX）
    ↓
你 審查並核准
    ↓
/ddd-tdd     紅燈 → 綠燈 → 驗收
    ↓
/ddd-doc     更新實作記錄，同步模組文檔
```

---

## 怎麼選流程

```
需求是什麼？
│
├─ 說不清楚         → /ddd-start → grill-me 先挖
├─ 改現有模組       → /ddd-start → grill-with-docs 驗證架構
├─ 全新功能         → /ddd-start → ddd-doc FXX
├─ 重構             → /ddd-start → zoom-out → ddd-doc RXX
├─ Bug              → /ddd-start → ddd-doc BXX
├─ 多個工作一次做   → /ddd-queue（每個 item 有明確需求與驗收方式）
└─ 大型改動         → /ddd-plan（後期細節要等前期結果才能定）
```

**queue 還是 plan？**
- 現在就能列出所有 item 的需求和驗收方式 → **queue**
- 後面要做什麼要等前面完成才知道 → **plan**

---

## 指令說明

### `/ddd-create-folder` — 初始化專案

在新專案中建立 DDD 所需的資料夾結構與初始模板。**第一次導入 DDD 時使用。**

```
documents/
├── ddd-email-notify.md         ← 通知信箱設定
├── implements/                 ← FXX / RXX / BXX
├── planning/                   ← PXX 多階段規劃書
├── queue/                      ← QXX 工作佇列
│   └── logs/
├── modules/                    ← 模組高層次文檔
└── guides/                     ← GXX 操作指南
CONTEXT.md                      ← 領域詞彙表（可選）
```

初始化後：填寫 `CONTEXT.md` 定義專案術語 → 填寫寄信設定 → `/ddd-start` 開始第一個工作。

---

### `/ddd-start` — 所有新工作的入口

偵測需求類型並路由到正確步驟，確保沒有模糊需求直接進入文檔撰寫。**每次提出新工作都從這裡開始。**

| 需求類型 | 路由到 |
|---|---|
| 模糊想法 | `grill-me` → `ddd-doc FXX` |
| 涉及現有架構的新功能 | `grill-with-docs` → `ddd-doc FXX` |
| 規格清晰的新功能 | `ddd-doc FXX` |
| 重構任務 | `zoom-out` → `ddd-doc RXX` |
| Bug 回報 | `ddd-doc BXX` |
| 2-5 個工作想一次做完 | `ddd-queue` |
| 多階段大型改動 | `ddd-plan` |

觸發 `ddd-plan` 的關鍵字（直接路由）：「幫我計劃」「規劃一下」「分階段規劃」「PXX」

---

### `/ddd-doc` — 起草與維護文檔

**編碼前：** 讀 `CONTEXT.md` → 確認需求清晰度 → 起草文檔（含驗收標準與測試場景）→ 交你審查。

**編碼後：** 更新實作記錄 → 審計模組文檔 → 詢問是否有架構技術債。

每份文檔包含：用戶故事、Given/When/Then 驗收標準、測試場景表、實作說明、**實作記錄**（完成後填入）。

---

### `/ddd-tdd` — 依文檔實作

**前提：必須有你核准的 FXX / RXX / BXX，否則停止。**

1. 讀文檔，提取驗收標準
2. **紅燈**：寫最小失敗測試，確認失敗
3. 若測試意外失敗 → 進 `diagnose` 結構化除錯，根因寫回文檔
4. **綠燈**：實作最小生產代碼
5. **驗收**：確認每個驗收標準都有測試覆蓋
6. 將實作記錄寫回文檔

---

### `/ddd-plan` — 大型改動規劃

將多階段需求拆成有序的執行計劃，起草 PXX 規劃書，每個階段對應一份 F/R/B 文檔。

**何時用：** 需求橫跨多個模組、後期細節依賴前期結果、或階段切分需要你審查。

**PXX 核准後逐階段執行：**
```
你說「繼續 PXX」→ AI 找最靠前未開始的階段 → 起草 F/R/B
→ 你核准 → ddd-tdd 實作 → 回填 PXX 狀態 → 重複直到完成
```

**範例：電商後台權限系統改版**

你說：「我要把後台的權限系統從角色制改成資源制，同時重構舊 admin API，最後做審核流程 UI。幫我規劃一下。」

AI 起草 `documents/planning/P01-permission-refactor.md`：

```
P01 — 後台權限系統改版

階段 1：重構舊 admin API（RXX）
  你的驗收方式：用現有帳號登入後台，所有功能正常，無回歸錯誤
  狀態：未開始

階段 2：實作資源制權限模型（FXX）
  你的驗收方式：可在後台設定「誰能對哪個資源做什麼動作」
  依賴：階段 1 完成
  狀態：未開始

階段 3：審核流程 UI（FXX）
  你的驗收方式：能看到待審核清單，點擊可核准或駁回並即時反映
  依賴：階段 2 完成
  狀態：未開始
```

你核准後，每次一個階段：
```
「請繼續 P01，推進下一個階段。」
→ AI 起草 R01 → 你核准 → ddd-tdd 實作 → 回填 P01
```

---

### `/ddd-queue` — 批次自動執行

讓 2-5 個已排序工作由 AI 逐一完成，每個 item 使用新 session，完成後獨立 commit。**想減少被打斷、讓 AI 連續工作時使用。**

**執行流程：**
1. 建立 QXX 文檔
2. 集中用 `grill-me` 一次釐清所有 item 的需求、驗收方式、停止條件
3. 你確認 QXX 後允許執行
4. AI 每個 item 開新 session，走 `ddd-start → ddd-doc → ddd-tdd`，完成後 commit
5. 遇到 blocked 立即停止並通知你

**範例一：三個獨立 Bug 修正**

你說：「我有三個 Bug 要修，需求都清楚了，排成 queue 一次跑完，blocked 寄信通知我。」

AI 建立 `documents/queue/Q01-bug-fixes.md`：

```yaml
batch_limit: 3
notify_email_from: bot@myproject.com
notify_email_to: you@example.com

items:
  - id: Q01-01
    title: 修正登入頁重複提交問題
    type: BXX
    clarification_status: clarified
    acceptance: 快速點擊登入按鈕兩次，只送出一次請求

  - id: Q01-02
    title: 修正訂單列表金額顯示錯誤
    type: BXX
    clarification_status: clarified
    acceptance: 訂單列表顯示的金額與訂單詳情頁一致

  - id: Q01-03
    title: 修正使用者頭像上傳後不刷新
    type: BXX
    clarification_status: clarified
    acceptance: 上傳頭像後頁面立即更新，不需重整
```

執行順序：
```
Q01-01：新 session → ddd-doc BXX → ddd-tdd → git commit → 停止
Q01-02：新 session → ddd-doc BXX → ddd-tdd → git commit → 停止
Q01-03：新 session → ddd-doc BXX → ddd-tdd → git commit → 停止
→ 整批完成，寄信通知你
```

**範例二：有依賴關係的 Queue**

你說：「先做用戶標籤功能，再做根據標籤篩選的搜尋，完成後通知我。」

```yaml
items:
  - id: Q01-01
    title: 用戶標籤功能
    type: FXX
    clarification_status: clarified
    acceptance: 可對用戶新增/刪除標籤，重整後仍存在

  - id: Q01-02
    title: 依標籤篩選用戶搜尋
    type: FXX
    depends_on: Q01-01
    unlock_condition: Q01-01 完成且標籤 API 可用
    clarification_status: clarified
    acceptance: 搜尋頁可選擇標籤篩選，結果只顯示有該標籤的用戶
```

Q01-01 完成並 commit 後，AI 自動解鎖 Q01-02 繼續。

---

### `/ddd-email-notify` — 通知設定

查看或設定 blocked / completed 時的寄信通知。

| 事件 | 是否寄信 |
|---|---|
| `/ddd-tdd` 單次完成 | ✓ |
| Queue 單個 item 完成 | ✗（由整批統一處理） |
| Queue 整批完成 | ✓ |
| Queue blocked | ✓ |

設定位置：`documents/ddd-email-notify.md`（專案層）或 QXX frontmatter（單一 queue）。

---

## 輔助指令

| 指令 | 用途 | 典型使用時機 |
|---|---|---|
| `/grill-me` | 深挖真實需求，逐一提問釐清邊界 | 需求模糊，無法寫出驗收條件 |
| `/grill-with-docs` | 對照現有領域模型驗證計劃 | 新功能涉及修改現有模組 |
| `/diagnose` | 結構化除錯：重現 → 假設 → 修復 → 回歸測試 | 測試意外失敗，原因不明 |
| `/zoom-out` | 繪製模組地圖，確認重構範疇 | 啟動重構前確認影響邊界 |
| `/handoff` | 將當前對話壓縮為交棒文件 | 長時間實作，需要切換 session |
| `/prototype` | 建立一次性原型驗證設計方向 | 高度不確定的功能，提交 FXX 前 |
| `/improve-codebase-architecture` | 尋找架構深化機會，輸出 RXX 候選 | 實作後發現技術債 |
| `/caveman` | 切換超精簡溝通模式，減少 token | 文檔審查反覆確認措辭時 |
| `/git-guardrails` | 阻止危險 git 指令 | 實作期間的安全保護 |

---

## 文檔類別

| 前綴 | 類型 | 存放位置 | 使用時機 |
|---|---|---|---|
| `FXX` | 功能規格 | `documents/implements/` | 新功能 |
| `RXX` | 重構規格 | `documents/implements/` | 不改變行為的結構改善 |
| `BXX` | Bug 修正規格 | `documents/implements/` | 重現並修正缺陷 |
| `PXX` | 多階段規劃書 | `documents/planning/` | 橫跨多模組的大型改動 |
| `QXX` | 工作佇列 | `documents/queue/` | 2-5 個可自動推進的工作 |
| `GXX` | 操作指南 | `documents/guides/` | 執行測試、打包等操作說明 |

模組文檔（`documents/modules/`）：高層次描述現有模組的職責、邊界、依賴與已知限制。隨代碼同步更新，代碼與文檔衝突時以代碼為準。

---

## 常見情境速查

| 情境 | 指令 |
|---|---|
| 新功能 | `/ddd-start` → 描述需求 → 核准 FXX → `/ddd-tdd` |
| 需求模糊 | `/ddd-start` → AI 自動 grill-me → 收斂後進 ddd-doc |
| 發現 Bug | `/ddd-start` → 描述現象與重現步驟 → 核准 BXX → `/ddd-tdd` |
| 重構 | `/ddd-start` → zoom-out 確認範疇 → 核准 RXX → `/ddd-tdd` |
| 多個工作批次完成 | `/ddd-start` → ddd-queue → 確認 QXX → 自動執行 |
| 大型分階段改動 | `/ddd-start` → 說「幫我規劃」→ 核准 PXX → 逐階段執行 |
| 新專案導入 DDD | `/ddd-create-folder` → 填 CONTEXT.md → `/ddd-start` |
