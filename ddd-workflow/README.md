# DDD × Pocock 強化工作流程

不要將整個 `ddd-workflow/` 資料夾加入 AI 上下文。請只載入被觸發的 `skills/*/SKILL.md`，並讓技能在需要時再讀取同資料夾內的模板或參考文件，避免一次消耗整包 token。

## 快速開始

1. 將 `ddd-workflow/skills/` 安裝或註冊為可觸發技能
2. 需要初始化專案時，只觸發 `/ddd-create-folder`
3. `/ddd-create-folder` 會從 `skills/ddd-create-folder/templates/` 複製 F00 / R00 / B00 / P00 / Q00 / G00 模板
4. 填寫專案根目錄的 `CONTEXT.md`，定義你的專案領域術語
5. 執行 `/ddd-start` 開始任何新工作；若要讓 AI 連續處理多個已排序工作，使用 `/ddd-queue`

## 開發循環

```
人類提出需求
      ↓
/ddd-start   偵測類型 → 路由到正確入口
      ↓
/ddd-doc     草擬 FXX / RXX / BXX 文檔
      ↓
人類審查並核准文檔
      ↓
/ddd-tdd     紅燈 → 綠燈 → 驗收 → 更新文檔
```

長時間工作佇列不一定走 PXX：

```
人類列出 2-5 個已排序工作
      ↓
/ddd-queue   建立 QXX queue 文件
      ↓
orchestrator 每個 item 開新 Codex / Claude Code session
      ↓
worker 執行 ddd-start → ddd-doc → ddd-tdd → git commit → 更新 queue
      ↓
Agent Communication Ledger 記錄派工 / 問題 / 回答 / 測試 / 接棒
      ↓
完成 batch limit 或 blocked 後停止
```

## DDD 技能（主流程）

| 技能 | 用途 |
|---|---|
| `/ddd-create-folder` | **新專案初始化**：建立 documents/ 資料夾與 F00 / R00 / B00 / P00 / Q00 / G00 模板 |
| `/ddd-start` | 任何新工作的入口，偵測類型並路由 |
| `/ddd-doc` | 建立與維護 FXX / RXX / BXX 文檔及模組文檔 |
| `/ddd-tdd` | 文檔驅動的 TDD 實作，整合結構化除錯 |
| `/ddd-plan` | 大型改動的 PXX 多階段規劃；適用於後續範疇需等前階段完成才知道 |
| `/ddd-queue` | 多個已排序工作連續執行；可獨立或明確相依，每個 item 新 session 並各自 commit，QXX 內保留跨 agent 溝通紀錄 |

## Pocock 技能（按需使用）

這些技能由 `/ddd-start`、`/ddd-doc`、`/ddd-tdd` 在適當時機自動建議，也可以手動觸發：

| 技能 | 用途 | 典型觸發時機 |
|---|---|---|
| `/grill-me` | 挖掘模糊需求 | 想法還不具體時 |
| `/grill-with-docs` | 對照現有架構驗證計劃 | 需求涉及現有模組時 |
| `/zoom-out` | 繪製模組地圖取得全局視角 | 重構範疇確認、進入陌生代碼時 |
| `/to-prd` | 將非正式描述轉為結構化草稿 | 有粗略想法但缺乏規格時 |
| `/diagnose` | 結構化除錯迴圈 | 測試意外失敗時 |
| `/improve-codebase-architecture` | 浮現架構技術債 | 實作後、重構規劃前 |
| `/handoff` | 壓縮會話狀態供下一個對話繼續 | 長時間實作週期 |
| `/caveman` | 壓縮溝通減少 token | 文檔審查反覆確認措辭時 |
| `/prototype` | 一次性探索驗證方案 | 高不確定性的功能設計時 |
| `/git-guardrails` | 阻止破壞性 git 命令 | 首次設定專案時 |

## 資料夾結構

```
ddd-workflow/
├── README.md                      ← 本文件
├── CONTEXT.md                     ← 填寫你的專案領域術語（必填）
├── WORKFLOW.md                    ← 工作流程完整說明
└── skills/
    ├── ddd-start/   ← 入口路由
    ├── ddd-doc/     ← 文檔管理
    ├── ddd-tdd/     ← TDD 實作
    ├── ddd-plan/    ← 大型改動規劃
    ├── ddd-queue/   ← 長時間工作佇列
    ├── ddd-create-folder/
    │   └── templates/ ← F00 / R00 / B00 / P00 / Q00 / G00 的唯一模板來源
    ├── grill-me/
    ├── grill-with-docs/
    ├── zoom-out/
    ├── to-prd/
    ├── diagnose/
    ├── improve-codebase-architecture/
    ├── handoff/
    ├── caveman/
    ├── prototype/
    └── git-guardrails/
```

## 你的專案文檔應放在哪裡

將實作文檔和模組文檔放在你的**專案根目錄**下：

```
你的專案/
├── documents/
│   ├── implements/   ← FXX / RXX / BXX 工作文檔
│   ├── planning/     ← PXX 多階段規劃書
│   ├── queue/        ← QXX 長時間工作佇列
│   └── modules/      ← 模組高層次文檔
└── CONTEXT.md        ← 從 ddd-workflow/CONTEXT.md 複製並填寫
```

模板唯一來源位於 `ddd-workflow/skills/ddd-create-folder/templates/`。不要在 `ddd-workflow/documents/implements/` 再維護第二份模板。
