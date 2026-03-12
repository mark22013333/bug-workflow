---
name: bug-setup
description: Bug Workflow 首次設定引導。自動偵測 Notion 資料庫、建立設定檔、設定專案路徑對應。當使用者提到「bug-setup」、「設定 bug workflow」、「初始化 bug」時觸發此 Skill。
---

# bug-setup — Bug Workflow 首次設定

互動式引導使用者完成 Bug Workflow Plugin 的初始設定，產出設定檔供其他 Skill 使用。

---

## 前置條件

- 已安裝 Notion MCP Server（Claude Code 可使用 `notion-search`、`notion-fetch` 等工具）
- 擁有 Notion Workspace 存取權限

---

## 流程

### 1. 決定設定檔位置並檢查是否已存在

**設定檔路徑規則**（依序檢查）：

1. 先檢查 `~/.claude-company/bug-workflow-config.md`（公司環境）
2. 再檢查 `~/.claude/bug-workflow-config.md`（個人環境）
3. 若都不存在 → 詢問使用者要儲存到哪裡：
   ```
   請選擇設定檔儲存位置：
   1. ~/.claude-company/bug-workflow-config.md（公司環境，推薦用於團隊共用 Notion Workspace）
   2. ~/.claude/bug-workflow-config.md（個人環境，推薦用於私人 Notion Workspace）
   ```
4. 若已存在 → 詢問使用者要「重新設定」還是「更新專案對應」

### 2. 偵測 Notion 資料庫

#### 2-1. 搜尋「任務追蹤工具」

使用 `notion-search` 搜尋 Workspace 中包含「任務追蹤」的資料庫。

找到後用 `notion-fetch` 取得該資料庫的 data-source URL（`collection://...`），擷取 Data Source ID。

驗證欄位是否齊全（任務名稱、狀態、任務類型、優先順序、環境、根因分類、修復分支、專案資料庫）：
- 齊全 → 記錄 ID，繼續
- 缺少欄位 → 列出缺少的欄位，詢問使用者是否要自動新增（使用 `notion-update-data-source`）

#### 2-2. 搜尋「Bug 知識庫」

搜尋包含「bug」、「處理方式」的資料庫。

找到後同樣擷取 Data Source ID 並驗證欄位。

若找不到 → 詢問使用者：
- A) 指定一個現有資料庫作為知識庫
- B) 跳過知識庫（`/bug-close` 結案時不同步知識庫）

#### 2-3. 搜尋「專案資料庫」

搜尋包含「專案」的資料庫。

找到後驗證是否有「本機路徑」欄位：
- 有 → 記錄 ID
- 沒有 → 自動新增「本機路徑」Text 欄位

### 3. 設定專案路徑對應

取得當前工作目錄（`pwd`），搜尋專案資料庫中的所有專案，列出清單：

```
偵測到你目前在：/Users/xxx/projects/my-project

請選擇要對應的 Notion 專案（或輸入 0 跳過）：
1. 專案 A（本機路徑：未設定）
2. 專案 B（本機路徑：/Users/xxx/projects/B）
3. 專案 C（本機路徑：未設定）
```

使用者選擇後，自動將 `pwd` 寫入該專案的「本機路徑」欄位。

### 4. 產出設定檔

以 `references/config.template.md` 為模板，填入偵測到的 ID 與對應資訊，寫入使用者在步驟 1 選擇的路徑。

### 5. 回傳結果

向使用者顯示：

```
Bug Workflow 設定完成！

已偵測到的資料庫：
  ✅ 任務追蹤工具：1d8a401b-...
  ✅ Bug 知識庫：bd132aa4-...
  ✅ 專案資料庫：f67699b6-...

已設定的專案對應：
  • 我的專案 → /Users/xxx/projects/my-project

設定檔位置：~/.claude-company/bug-workflow-config.md

現在可以使用：
  /bug-start <問題簡述>     — 建立 Bug 條目
  /bug-update <內容>        — 更新調查資訊
  /bug-close                — 結案並同步知識庫
  /bug-search <關鍵字>      — 搜尋過往 Bug 解法
  /bug-update reopen <Bug>  — 重新開啟已結案 Bug
```

---

## 邊界情況

- **Notion MCP 未安裝**：提示使用者先安裝 Notion Plugin（`claude plugin install Notion`）
- **Workspace 中有多個類似資料庫**：列出候選讓使用者選擇
- **使用者想新增更多專案對應**：可重複執行 `/bug-setup`，選擇「更新專案對應」
- **設定檔被意外刪除**：重新執行 `/bug-setup` 即可重建
