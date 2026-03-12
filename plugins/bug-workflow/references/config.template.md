# Bug Workflow 設定檔

此檔案由 `/bug-setup` 自動產生，儲存於 `~/.claude-company/bug-workflow-config.md`。
所有 Bug Skill 會讀取此檔案取得 Notion ID 與專案對應。

---

## Notion 資料庫 Data Source ID

| 資料庫 | Data Source ID | 用途 |
|--------|---------------|------|
| 任務追蹤工具 | `<待填入>` | Bug 生命週期管理（主要） |
| Bug 知識庫 | `<待填入>` | Bug 精簡索引（結案同步） |
| 專案資料庫 | `<待填入>` | 專案 Relation 來源 |

## 專案路徑對應

Skill 透過 `pwd` 取得當前工作目錄，以前綴匹配方式自動對應 Notion 專案。

| Notion 專案名稱 | 本機路徑 | 說明 |
|----------------|----------|------|
| （範例）我的專案 | `/Users/xxx/projects/my-project` | 範例，請替換 |

## 欄位對照

### 任務追蹤工具

| 欄位 | 類型 | 選項值 |
|------|------|--------|
| 任務名稱 | Title | — |
| 狀態 | Status | `未開始` / `進行中` / `測試中` / `已完成` |
| 任務類型 | Multi-select | `🐞 錯誤` / `💬 功能要求` / `💅 細調` |
| 優先順序 | Select | `高` / `中` / `低` |
| 環境 | Select | `測試` / `UAT` / `正式` |
| 根因分類 | Select | `邏輯錯誤` / `資料異常` / `設定問題` / `第三方API` / `效能` / `權限` / `前端UI` |
| 修復分支 | Text | Git branch 名稱 |
| 專案資料庫 | Relation | 關聯至專案資料庫 |

### Bug 知識庫

| 欄位 | 類型 | 選項值 |
|------|------|--------|
| Name | Title | — |
| Tags | Multi-select | 依專案自訂（如 `SSO` / `推播` / `排程` 等） |
| 難易度 | Select | `普通(2~4h)` / `困難(4~6h)` |
| 參考連結 | URL | — |
| 專案資料庫 | Relation | 關聯至專案資料庫 |
