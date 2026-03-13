---
name: feature-spec
description: 使用 Agent(opus) 分析需求並產出完整技術規格書，寫入 Notion 頁面。當使用者提到「feature-spec」、「寫規格」、「規格書」、「技術規格」時觸發此 Skill。
---

# feature-spec — 技術規格書產出

使用 Agent（Opus 模型）分析需求描述與專案上下文，產出完整的技術規格書，並寫入 Notion 功能頁面的「📐 技術規格」區塊。

---

## 設定檔

執行前依序檢查以下路徑，讀取第一個找到的設定檔：
1. `~/.claude-company/feature-workflow-config.md`（公司環境）
2. `~/.claude/feature-workflow-config.md`（個人環境）

若都不存在，提示使用者先執行 `/feature-setup`。

---

## 前置條件

- 已使用 `/feature-start` 建立功能條目
- Notion 頁面已有「📋 需求描述」區塊的基本內容

---

## 流程

### 1. 定位目標功能 Notion 頁面

使用與 feature-update 相同的定位邏輯：
1. 讀取設定檔，取得「任務追蹤工具」Data Source ID
2. 取得 Git Repo 識別碼（從 `git remote get-url origin` 解析）和 Git branch
3. 搜尋 Notion 中狀態為「進行中」、任務類型為「💬 功能要求」的條目
4. 優先匹配：Git branch → 單一候選 → 專案篩選 → 互動式選擇

### 2. 取得需求描述

使用 `notion-fetch` 取得目標頁面完整內容，擷取「📋 需求描述」區塊。

若使用者在指令中提供了補充需求（如 `/feature-spec 還需要支援批次匯出`），合併到需求描述中。

### 3. 讀取專案上下文

收集以下資訊作為 Agent 的輸入：

#### 3-1. 專案 CLAUDE.md

讀取 `pwd` 下最近的 CLAUDE.md（向上搜尋），取得：
- 技術棧描述
- 架構模式和分層規則
- 命名慣例
- 例外處理慣例
- API 回傳格式慣例

#### 3-2. 技術棧資訊

從設定檔的「專案對應」表取得當前專案的技術棧 ID。

#### 3-3. 現有類似功能程式碼

使用 Glob/Grep 掃描專案中與需求相關的現有 Controller 和 Service，取得 1-2 個作為 API 風格參考：
- Controller 的 `@RequestMapping` 模式
- Service 方法的參數和回傳值模式
- 錯誤處理模式

### 4. 啟動規格分析 Agent

使用 **Agent tool** 啟動 subagent，參數：
- **model**: `opus`
- **subagent_type**: 若使用者已安裝獨立 Agent，可使用 `feature-spec-analyst`；否則使用內嵌指示
- **prompt**: 組合以下內容傳入 Agent：

```
你是一位資深技術規格分析師。

## 專案上下文
{專案 CLAUDE.md 內容}

## 技術棧
{技術棧 ID 和描述}

## 現有 API 風格參考
{現有 Controller/Service 程式碼片段}

## 需求描述
{從 Notion 取得的需求描述}

## 補充需求
{使用者額外提供的補充}

## 任務
根據以上資訊，產出完整的技術規格，包含：
1. 功能範圍（In Scope / Out of Scope）
2. API 端點設計（表格格式：端點、Method、請求參數/Body、回傳格式、說明）
3. 業務邏輯規則（每個 API 的核心邏輯、前置驗證、處理流程、邊界條件）
4. 錯誤處理策略（依專案慣例，表格格式：錯誤場景、處理方式、HTTP 狀態碼）
5. 分層決策（哪些邏輯放哪一層，依據 CLAUDE.md 的架構規則）
6. 效能需求（預期資料量、分頁、快取需求）

API 路徑、回傳格式、錯誤處理必須遵循專案既有慣例。
輸出使用繁體中文，Markdown 格式。
```

### 5. 寫入 Notion

Agent 完成後，擷取其輸出：

1. 使用 `notion-update-page` 的 `update_content`，將規格內容寫入「📐 技術規格」區塊
   - 如果區塊原本只有空的子標題，替換整個區塊內容
   - 如果區塊已有內容，在末尾附加並標記更新時間

2. 使用 `notion-update-page` 更新 Property：
   - 開發階段 = `規格設計`

### 6. 回傳結果

向使用者顯示：
- Agent 產出的規格摘要（API 數量、主要邏輯）
- Notion 頁面連結
- 提示後續指令：
  ```
  技術規格已寫入 Notion！後續可使用：
  • /feature-db    — 根據規格設計資料庫
  • /feature-arch  — 根據規格設計架構
  • /feature-update — 補充或修改規格
  ```

---

## 邊界情況

- **需求描述為空**：提示使用者先在 Notion 頁面填寫需求，或在指令中直接提供
- **專案無 CLAUDE.md**：使用掃描到的程式碼結構推斷，並提醒使用者建立 CLAUDE.md
- **Agent 輸出過長**：截斷至合理長度，提供完整版連結
- **使用者想修改規格**：可再次執行 `/feature-spec` 或用 `/feature-update API 調整：...` 局部更新
