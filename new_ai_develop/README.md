# DDD × Pocock 強化工作流程

將此資料夾加入 AI 的上下文，即可使用完整的 DDD 強化工作流程。

## 快速開始

1. 將整個 `new_ai_develop/` 資料夾加入 AI 對話（或複製到你的專案）
2. 執行 `/ddd-create-folder` 建立專案文檔結構與模板
3. 填寫 `CONTEXT.md`，定義你的專案領域術語
4. 執行 `/ddd-start` 開始任何新工作

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

## DDD 技能（主流程）

| 技能 | 用途 |
|---|---|
| `/ddd-create-folder` | **新專案初始化**：建立 documents/ 資料夾與 F00 / R00 / B00 模板 |
| `/ddd-start` | 任何新工作的入口，偵測類型並路由 |
| `/ddd-doc` | 建立與維護 FXX / RXX / BXX 文檔及模組文檔 |
| `/ddd-tdd` | 文檔驅動的 TDD 實作，整合結構化除錯 |

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
new_ai_develop/
├── README.md                      ← 本文件
├── CONTEXT.md                     ← 填寫你的專案領域術語（必填）
├── NEW_DDD_SKILLs.md              ← 工作流程完整說明
├── documents/
│   └── implements/
│       ├── F00-功能需求書模板.md   ← FXX 模板
│       ├── R00-重構任務模板.md     ← RXX 模板
│       └── B00-Bug修正模板.md      ← BXX 模板
└── skills/
    ├── ddd-start/   ← 入口路由
    ├── ddd-doc/     ← 文檔管理
    ├── ddd-tdd/     ← TDD 實作
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
│   └── modules/      ← 模組高層次文檔
└── CONTEXT.md        ← 從 new_ai_develop/CONTEXT.md 複製並填寫
```
