---
name: feature-db
description: 使用 Agent(opus) 根據技術規格設計資料庫結構，產出 CREATE TABLE / INDEX / 範例資料 / Rollback SQL，寫入 Notion 頁面。當使用者提到「feature-db」、「設計資料庫」、「DB 設計」、「建表」時觸發此 Skill。
---

# feature-db — 資料庫設計

使用 Agent（Opus 模型）根據技術規格與專案上下文，設計資料表結構、索引和遷移 SQL，並寫入 Notion 功能頁面的「🗄️ 資料庫設計」區塊。

---

## 設定檔

執行前依序檢查以下路徑，讀取第一個找到的設定檔：
1. `~/.claude-company/feature-workflow-config.md`（公司環境）
2. `~/.claude/feature-workflow-config.md`（個人環境）

若都不存在，提示使用者先執行 `/feature-setup`。

---

## 前置條件

- 已使用 `/feature-start` 建立功能條目
- 建議已執行 `/feature-spec` 產出技術規格（非強制）

---

## 流程

### 1. 定位目標功能 Notion 頁面

使用與 feature-update 相同的定位邏輯。

### 2. 取得技術規格

使用 `notion-fetch` 取得目標頁面完整內容，擷取「📐 技術規格」區塊。

若技術規格為空，提示使用者先執行 `/feature-spec` 或手動填寫，但不強制阻擋（使用者可提供口頭描述）。

若使用者在指令中提供了補充說明（如 `/feature-db 需要支援軟刪除`），合併到設計輸入中。

### 3. 讀取專案上下文

收集以下資訊作為 Agent 的輸入：

#### 3-1. 專案 CLAUDE.md

讀取 CLAUDE.md，識別：
- DB 類型（MSSQL / MySQL / PostgreSQL）
- 資料庫連線資訊中的 DB 類型提示

#### 3-2. 現有 Entity/POJO 慣例

使用 Glob 掃描現有 Entity/POJO（取 2-3 個範本）：
- 命名慣例（表名、欄位名）
- 公共欄位（creator、create_time 等）
- 註解風格（`@Table`、`@Column`、`@Id`）
- Lombok 使用情況

#### 3-3. 資料庫規範

讀取 `~/.claude/rules/database.md`（若存在），取得命名規範、索引設計規範、資料型別規範。

### 4. 啟動 DB 設計 Agent

使用 **Agent tool** 啟動 subagent，參數：
- **model**: `opus`
- **prompt**: 組合以下內容傳入 Agent：

```
你是一位資深資料庫設計師。

## 專案上下文
{專案 CLAUDE.md 中 DB 相關描述}

## DB 類型
{MSSQL / MySQL / PostgreSQL}

## 現有 Entity 慣例
{現有 Entity/POJO 範本程式碼}

## 公共欄位模式
{從現有 Entity 識別的公共欄位}

## 資料庫規範
{~/.claude/rules/database.md 內容，若存在}

## 技術規格
{從 Notion 取得的技術規格}

## 補充說明
{使用者額外提供的補充}

## 任務
根據以上資訊，設計資料庫結構，包含：
1. CREATE TABLE（使用正確的 DB 語法，含欄位註解和公共欄位）
2. CREATE INDEX（遵循專案命名慣例）
3. 範例資料 INSERT（3-5 筆）
4. Rollback SQL（DROP TABLE / DROP INDEX）

若有修改現有表的需求，提供 ALTER TABLE 語句。
命名慣例、資料型別、索引風格必須與現有表一致。
輸出使用繁體中文，Markdown 格式，SQL 使用 code block。
```

### 5. 寫入 Notion

Agent 完成後：

1. 使用 `notion-update-page` 的 `update_content`，將 DB 設計內容寫入「🗄️ 資料庫設計」區塊

2. 使用 `notion-update-page` 更新 Property：
   - 開發階段 = `DB 設計`

### 6. 詢問是否輸出 SQL 檔

```
是否將 SQL 輸出到專案目錄？
1. 是，輸出到 doc/{功能名稱}.sql
2. 是，自訂路徑
3. 否，僅保留在 Notion
```

若選擇輸出，使用 Write tool 建立 SQL 檔案（包含部署 SQL 和 Rollback SQL）。

### 7. 回傳結果

向使用者顯示：
- 設計摘要（新增 N 個表、M 個索引）
- Notion 頁面連結
- SQL 檔案路徑（若有輸出）
- 提示後續指令：
  ```
  DB 設計已寫入 Notion！後續可使用：
  • /feature-arch      — 根據規格 + DB 設計架構
  • /feature-scaffold   — 直接產生程式碼骨架
  • /feature-update     — 調整 DB 設計
  ```

---

## 邊界情況

- **技術規格為空**：提示建議先執行 `/feature-spec`，但允許使用者口頭描述需求
- **無法判斷 DB 類型**：詢問使用者確認
- **專案無 Entity 範本**：使用資料庫規範（database.md）的預設慣例
- **使用者想修改 DB 設計**：可再次執行 `/feature-db` 或用 `/feature-update DB 調整：...` 局部更新
