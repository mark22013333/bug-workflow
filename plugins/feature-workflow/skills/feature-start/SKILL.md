---
name: feature-start
description: 在 Notion「任務追蹤工具」建立功能需求條目並填入標準化模板，可選建立 Git feature branch。當使用者提到「feature-start」、「新功能」、「建立功能」、「開始開發」時觸發此 Skill。
---

# feature-start — 建立功能需求條目

在 Notion「任務追蹤工具」資料庫建立一筆功能需求條目，自動填入標準化頁面模板（7 個區塊），關聯對應專案，並可選建立 Git feature branch。

---

## 設定檔

執行前依序檢查以下路徑，讀取第一個找到的設定檔：
1. `~/.claude-company/feature-workflow-config.md`（公司環境）
2. `~/.claude/feature-workflow-config.md`（個人環境）

若都不存在，提示使用者先執行 `/feature-setup` 完成初始設定。

---

## 流程

### 1. 解析使用者輸入

使用者會以以下格式觸發：

```
/feature-start <功能簡述>
```

從使用者輸入中擷取：
- **功能簡述**（必填）：作為「任務名稱」

### 2. 偵測環境資訊（自動專案對應）

取得 branch 名稱、當前工作目錄及 Git Repo 識別碼：

```bash
# 分支名稱
git branch --show-current 2>/dev/null || echo ""

# 當前工作目錄
pwd

# Git 遠端 URL（用於自動對應 Notion 專案）
git remote get-url origin 2>/dev/null || echo ""
```

Git Repo 識別碼解析規則：
- 從 `git remote get-url origin` 取得遠端 URL（支援 HTTPS / SSH）
- 去除 `.git` 後綴
- Git host 含 `intumit`（公司 GitLab）→ 只取 `{group}/{repo}`，例如 `FUB03P2402/PushAPIService`
- 其他（GitHub 等）→ 加上 host：`{host}/{group}/{repo}`，例如 `github.com/mark22013333/crew`

**自動專案對應邏輯**：

1. 執行 `git remote get-url origin` 取得 Git 遠端 URL
2. 解析為 Git Repo 識別碼（host 含 `intumit` → `{group}/{repo}`，其他 → `{host}/{group}/{repo}`，去除 `.git` 後綴）
3. 讀取設定檔中「專案對應」表，精確匹配「Git Repo」欄位
4. 若匹配成功 → 自動選定該專案，不再詢問
5. 若不在 Git repo 或匹配失敗 → 進入互動式選擇

若設定檔中無對應，也可用 `notion-search` 搜尋 Notion「專案資料庫」（Data Source ID 見設定檔），找「Git Repo」欄位與當前識別碼匹配的專案。

### 3. 互動式補充資訊

若使用者未在初始輸入中提供以下資訊，依序詢問：

1. **所屬專案**（若自動偵測失敗）：搜尋 Notion「專案資料庫」，列出「進行中」的專案供選擇
2. **優先順序**（預設「中」）：`高` / `中` / `低`
3. **難度**（預設「中」）：`小` / `中` / `大`

使用者可在初始輸入中直接指定，例如：
```
/feature-start 新增訂閱推播統計報表 高 大
```

### 4. 建立 Notion 條目

使用 `notion-create-pages` 在「任務追蹤工具」資料庫建立新條目：

**Data Source ID**：從設定檔的「任務追蹤工具」取得

**Properties**：

| 欄位 | 值 |
|------|-----|
| 任務名稱 | 使用者提供的功能簡述 |
| 任務類型 | `["💬 功能要求"]` |
| 狀態 | `進行中` |
| 優先順序 | 使用者選擇（預設「中」） |
| 難度 | 使用者選擇（預設「中」） |
| 開發階段 | `需求分析` |
| 修復分支 | Git branch 名稱（若後續建立）|
| 專案資料庫 | 關聯的專案頁面 URL |

### 5. 填入頁面模板

頁面的 content 使用以下標準模板（參考 `references/notion-page-template.md`）：

```
## 📋 需求描述
- **功能目的**：
- **目標使用者**：
- **業務價值**：
- **使用情境**：
  1. ...
- **驗收條件**：
  - [ ] ...

---
## 📐 技術規格
### 功能範圍（In Scope / Out of Scope）
### API 端點設計
### 業務邏輯規則
### 錯誤處理
### 效能需求

---
## 🗄️ 資料庫設計
### 新增表格
### 修改表格
### 索引設計
### 遷移 SQL

---
## 🏗️ 架構設計
### 分層架構圖
### 類別/介面清單
### 設計模式
### 依賴關係

---
## 📁 程式碼清單
### 新增檔案
### 修改檔案

---
## 🧪 測試計畫
- [ ] 單元測試
- [ ] 整合測試
- [ ] API 測試
- [ ] 前端功能測試

---
## 📝 開發日誌
### [{日期}] 開始開發
```

若使用者在初始輸入中已提供功能描述，將其預填入「需求描述」區塊的「功能目的」欄位。

### 6. 詢問是否建立 Git feature branch

```
是否建立 Git feature branch？
1. 是，建立 feature/{功能簡述英文簡寫}（從當前分支）
2. 是，自訂分支名稱
3. 否，稍後再建立
```

若選擇建立：
1. 執行 `git checkout -b feature/{branch-name}`
2. 將分支名稱回填到 Notion 條目的「修復分支」欄位

### 7. 回傳結果

向使用者回傳：
- Notion 頁面連結
- 建立的條目摘要（任務名稱、專案、優先順序、難度、開發階段）
- Git branch 名稱（若有建立）
- 提示後續可用指令：
  ```
  功能條目已建立！後續可依序使用：
  • /feature-spec [補充需求]      — 產出技術規格書（Agent 分析）
  • /feature-db [補充說明]        — 設計資料庫（Agent 設計）
  • /feature-arch [補充說明]      — 設計架構（Agent 設計）
  • /feature-scaffold [--dry-run] — 產生程式碼骨架（Agent 產生）
  • /feature-update <進度>        — 隨時記錄開發進度

  流程非強制線性，可跳過或反覆執行任何步驟。
  ```

---

## 邊界情況

- **設定檔不存在**：提示使用者先執行 `/feature-setup` 完成初始設定
- **不在 Git repo 中**：跳過分支與專案自動偵測，修復分支留空；進入互動式選擇專案
- **使用者未指定專案**：列出進行中的專案供選擇；若只有一個專案則自動選定
- **Notion API 失敗**：顯示錯誤訊息，建議使用者手動在 Notion 建立
- **分支名稱衝突**：提示使用者自訂名稱或切換到現有分支
