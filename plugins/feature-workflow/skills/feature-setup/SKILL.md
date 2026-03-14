---
name: feature-setup
description: Feature Workflow 首次設定引導。自動偵測 Notion 資料庫、匯入 bug-workflow 共用 ID、設定專案對應與技術棧、可選安裝獨立 Agent。當使用者提到「feature-setup」、「設定 feature workflow」、「初始化 feature」時觸發此 Skill。
---

# feature-setup — Feature Workflow 首次設定

互動式引導使用者完成 Feature Workflow Plugin 的初始設定，產出設定檔供其他 Skill 使用。

---

## 前置條件

- 已安裝 Notion MCP Server（Claude Code 可使用 `notion-search`、`notion-fetch` 等工具）
- 擁有 Notion Workspace 存取權限

---

## 流程

### 1. 決定設定檔位置並檢查是否已存在

**設定檔路徑規則**（依序檢查）：

1. 先檢查 `~/.claude-company/feature-workflow-config.md`（公司環境）
2. 再檢查 `~/.claude/feature-workflow-config.md`（個人環境）
3. 若都不存在 → 詢問使用者要儲存到哪裡：
   ```
   請選擇設定檔儲存位置：
   1. ~/.claude-company/feature-workflow-config.md（公司環境，推薦用於團隊共用 Notion Workspace）
   2. ~/.claude/feature-workflow-config.md（個人環境，推薦用於私人 Notion Workspace）
   ```
4. 若已存在 → 詢問使用者要「重新設定」還是「更新專案對應」

### 2. 檢查並匯入 bug-workflow 共用 ID

檢查 bug-workflow 設定檔是否存在（依序：`~/.claude-company/bug-workflow-config.md`、`~/.claude/bug-workflow-config.md`）。

若找到 bug-workflow 設定檔：
1. 擷取「任務追蹤工具」Data Source ID → 直接匯入（兩個 workflow 共用同一個任務追蹤工具）
2. 擷取「專案資料庫」Data Source ID → 直接匯入
3. 擷取「專案對應」表 → 作為基礎，後續補充技術棧欄位
4. 向使用者顯示：
   ```
   已從 bug-workflow 匯入共用設定：
     ✅ 任務追蹤工具：1d8a401b-...
     ✅ 專案資料庫：f67699b6-...
     ✅ 專案對應：2 個專案
   ```

若未找到 → 進入步驟 3 完整設定。

### 3. 偵測 Notion 資料庫

#### 3-1. 搜尋/驗證「任務追蹤工具」

若已從 bug-workflow 匯入，使用 `notion-fetch` 驗證並檢查「開發階段」欄位：
- 若「開發階段」欄位已存在 → 繼續
- 若不存在 → 使用 `notion-update-data-source` 新增「開發階段」Select 欄位，選項值：
  `需求分析` / `規格設計` / `DB 設計` / `架構設計` / `開發中` / `程式碼審查` / `測試中`

若未從 bug-workflow 匯入，使用 `notion-search` 搜尋 Workspace 中包含「任務追蹤」的資料庫：
- 找到後用 `notion-fetch` 取得 data-source URL，擷取 Data Source ID
- 驗證欄位是否齊全（任務名稱、狀態、任務類型、優先順序、難度、開發階段、修復分支、專案資料庫）
- 缺少欄位 → 列出並詢問是否自動新增

同時檢查「難度」欄位（bug-workflow 可能未建立）：
- 若「難度」欄位不存在 → 使用 `notion-update-data-source` 新增「難度」Select 欄位，選項值：`小` / `中` / `大`

#### 3-2. 搜尋「功能設計庫」

搜尋包含「功能」、「設計」的資料庫。

找到後擷取 Data Source ID 並驗證欄位（Name、Tags、設計類型、技術棧、參考連結、專案資料庫）：
- 欄位齊全 → 記錄 ID
- 缺少欄位 → 詢問是否自動新增

若找不到 → 詢問使用者：
- A) 建立新的「功能設計庫」資料庫（inline 模式，含有色 Tags）
- B) 指定一個現有資料庫作為功能設計庫
- C) 跳過設計庫（`/feature-close` 結案時不同步設計庫）

**建立功能設計庫**時（選項 A），使用 `notion-create-database`：
- 父頁面：與「專案資料庫」同級（通常是「⇒⇒ 公司專案」）
- 設為 **inline 模式**（`is_inline: true`），讓資料庫在頁面中直接顯示表格
- Schema：
  ```sql
  CREATE TABLE (
    "Name" TITLE,
    "Tags" MULTI_SELECT('API':blue, '報表':purple, '標籤':green, '推播':orange, '排程':yellow, '綁定':pink, '權限':red, 'Rich Menu':brown, '匯出匯入':gray, 'Flex Message':default),
    "設計類型" SELECT('規格書':blue, 'DB 設計':yellow, '架構設計':orange, '完整設計':green),
    "技術棧" SELECT('spring-mvc-mybatis':blue, 'spring-boot-mybatis':green, 'spring-boot-jpa':purple, 'spring-boot-mybatis-plus':orange, 'spring-mvc-jpa':red),
    "參考連結" URL,
    "專案資料庫" RELATION('{專案資料庫 data_source_id}')
  )
  ```
- 建立後使用 `notion-update-data-source` 設定 `is_inline: true`

#### 3-3. 偵測或建立「專案資料庫」

若已從 bug-workflow 匯入 → 使用 `notion-fetch` 驗證並補齊欄位（見下方欄位表），特別確認「技術棧」欄位存在。

若未匯入 → 使用 `notion-search` 搜尋包含「專案」的資料庫。

**專案資料庫標準欄位**：

| 欄位 | 類型 | 說明 | 必要性 |
|------|------|------|--------|
| 專案名稱 | Title | 顯示名稱 | 必要 |
| Git Repo | Text | Git 遠端倉庫識別碼（如 `FUB03P2402/PushAPIService`） | 必要 |
| 技術棧 | Select | scaffold 用（`spring-mvc-mybatis` 等） | 必要（feature-workflow） |
| SIT 主機 | Text | SIT 部署主機資訊 | 選用 |
| UAT 主機 | Text | UAT 部署主機資訊 | 選用 |
| 正式環境主機 | Text | 正式環境部署主機資訊 | 選用 |
| 部署方式 | Text | 部署指令或流程簡述 | 選用 |
| 狀態 | Status | `進行中` / `維護中` / `已結案` | 建議 |
| 說明 | Text | 專案簡要描述 | 選用 |

**情境 A：找到現有資料庫** → 驗證欄位，缺少的詢問是否新增。
**情境 B：找不到** → 詢問使用者建立新的或指定現有（流程同 bug-setup 的 2-3）。

### 4. 設定專案對應

> **專案新增/偵測邏輯統一由 `/project-add` 處理**，避免重複維護 Git 偵測、技術棧偵測、Notion 建立等邏輯。

#### 4-1. 匯入 bug-workflow 已有的專案對應

**若已從 bug-workflow 匯入專案對應**：

直接將匯入的專案列複製到 feature-workflow 設定檔（技術棧欄位暫時留空）：
```
已從 bug-workflow 匯入 2 個專案對應（技術棧待補充）：
  • 北市府-TPE01P2101 → TPE01P2101/LineBC [技術棧：待設定]
  • FIA01P2403 WCS → FUB03P2402/WCS [技術棧：待設定]
```

**若未匯入** → 專案對應表為空，將在步驟 4-2 引導新增。

#### 4-2. 委託 `/project-add` 新增或補充當前專案

設定檔產出後（步驟 6），自動詢問使用者：

```
是否立即新增當前專案到設定檔？
  → 將執行 /project-add 引導您完成專案偵測、技術棧設定與 Notion 同步。
  [Y/n]
```

- 若選擇 **是** → 執行 `/project-add` 流程（自動偵測 Git Repo、技術棧、搜尋/建立 Notion 專案、同步所有設定檔）
- 若選擇 **否** → 跳過，使用者可稍後手動執行 `/project-add`
- 若已從 bug-workflow 匯入且當前專案已在列表中 → `/project-add` 會偵測到已存在，進入「更新」模式補充技術棧

### 5. Agent 安裝（選用）

詢問使用者是否安裝獨立 Agent 定義檔：

```
是否安裝獨立 Agent？（安裝後可在任何對話中直接使用，不限於 feature-workflow Skill）

1. 是，安裝到 ~/.claude-company/agents/（公司環境）
2. 是，安裝到 ~/.claude/agents/（個人環境）
3. 否，跳過（僅使用 Skill 內嵌 Agent 模式）

Agent 清單：
  • feature-spec-analyst — 規格分析師（需求拆解、API 設計）
  • feature-db-designer — DB 設計師（表結構、索引、遷移 SQL）
  • feature-backend-designer — 架構設計師（分層設計、設計模式）
  • feature-code-generator — 程式碼產生器（按專案慣例產生程式碼）
```

若選擇安裝：
1. 讀取 plugin 目錄下 `agents/` 的 4 個 `.md` 檔案
2. 複製到使用者選擇的目標目錄
3. 顯示安裝結果

### 6. 產出設定檔

以 `references/config.template.md` 為模板，填入偵測到的 ID、專案對應與技術棧資訊，寫入使用者在步驟 1 選擇的路徑。

### 7. 回傳結果

向使用者顯示：

```
Feature Workflow 設定完成！

已偵測到的資料庫：
  ✅ 任務追蹤工具：1d8a401b-...（含「開發階段」欄位）
  ✅ 功能設計庫：xxxxxxxx-...
  ✅ 專案資料庫：f67699b6-...

Agent 安裝：已安裝 4 個 Agent 到 ~/.claude-company/agents/

設定檔位置：~/.claude-company/feature-workflow-config.md

專案管理：
  /project-add                  — 新增或更新專案（Git 偵測 + 技術棧 + Notion 同步）

功能開發：
  /feature-start <功能簡述>     — 建立功能需求
  /feature-spec [補充需求]      — 產出技術規格書
  /feature-db [補充說明]        — 設計資料庫
  /feature-arch [補充說明]      — 設計架構
  /feature-scaffold [--dry-run] — 產生程式碼骨架
  /feature-update <進度>        — 更新開發進度
  /feature-review               — 程式碼品質檢查
  /feature-close                — 結案
```

---

## 邊界情況

- **Notion MCP 未安裝**：提示使用者先安裝 Notion Plugin（`claude plugin install Notion`）
- **Workspace 中有多個類似資料庫**：列出候選讓使用者選擇
- **bug-workflow 設定檔部分資訊不全**：僅匯入有效的 ID，其餘進入完整偵測流程
- **使用者想新增更多專案對應**：引導使用 `/project-add`（不需重複執行 `/feature-setup`）
- **設定檔被意外刪除**：重新執行 `/feature-setup` 即可重建
- **Agent 目標目錄已有同名檔案**：詢問是否覆蓋
- **專案資料庫已有欄位但名稱不同**（如「Repo」vs「Git Repo」）：列出現有欄位讓使用者選擇對應
- **不在 Git repo 中**：`/project-add` 會處理此情境（互動式選擇或手動輸入）
- **Git remote 不存在**：`/project-add` 會處理此情境（Git Repo 留空，稍後補填）
