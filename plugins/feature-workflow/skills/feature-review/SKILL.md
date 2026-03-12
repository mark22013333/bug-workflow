---
name: feature-review
description: 根據專案 CLAUDE.md 動態產生檢查項目，對功能開發的程式碼進行品質審查並產出報告。當使用者提到「feature-review」、「程式碼審查」、「review」、「檢查品質」時觸發此 Skill。
---

# feature-review — 程式碼品質檢查

根據專案 CLAUDE.md 和 Git diff，對功能開發的程式碼進行品質審查，產出審查報告並寫入 Notion 功能頁面。

---

## 設定檔

執行前依序檢查以下路徑，讀取第一個找到的設定檔：
1. `~/.claude-company/feature-workflow-config.md`（公司環境）
2. `~/.claude/feature-workflow-config.md`（個人環境）

若都不存在，提示使用者先執行 `/feature-setup`。

---

## 前置條件

- 已使用 `/feature-start` 建立功能條目
- 有已 commit 或 staged 的程式碼變更

---

## 流程

### 1. 定位目標功能 Notion 頁面

使用與 feature-update 相同的定位邏輯。

### 2. 取得架構設計（預期類別清單）

使用 `notion-fetch` 取得目標頁面，擷取「🏗️ 架構設計」區塊中的「類別/介面清單」。

這份清單用於對照實際產出，檢查是否有遺漏或多餘的類別。

### 3. 讀取專案上下文

#### 3-1. 專案 CLAUDE.md

讀取 CLAUDE.md，取得品質規範、命名慣例、架構規則。

#### 3-2. 相關 rules 檔案

讀取以下 rules（若存在）：
- `~/.claude/rules/java-performance.md` → 效能相關檢查
- `~/.claude/rules/java-concurrency.md` → 執行緒安全檢查
- `~/.claude/rules/database.md` → 資料庫相關檢查
- `~/.claude/rules/design-patterns.md` → 設計模式檢查

### 4. 取得 Git diff

```bash
# 取得功能分支的所有變更（相對於基礎分支）
git diff $(git merge-base HEAD production)..HEAD --stat
git diff $(git merge-base HEAD production)..HEAD
```

若無法判斷基礎分支，使用最近 N 個 commit 的 diff。

### 5. 執行品質審查

依專案 CLAUDE.md **動態產生**檢查項目，分為兩類：

#### 通用檢查（所有專案適用）

| 類別 | 檢查項目 |
|------|---------|
| 方法長度 | 方法超過 50-80 行 → WARNING |
| 巢狀層級 | 超過 3 層巢狀 → WARNING |
| 命名一致性 | 命名不符合專案慣例 → WARNING |
| 硬編碼常數 | 魔術數字或硬編碼字串 → WARNING |
| 執行緒安全 | 共享可變狀態未保護 → ERROR |
| 例外處理 | 空 catch block、過寬的 Exception → WARNING |
| SQL 注入 | 字串拼接 SQL → ERROR |
| 日誌品質 | catch 後未記錄 log → INFO |

#### 專案特有檢查（從 CLAUDE.md 提取）

根據 CLAUDE.md 中描述的架構規則動態產生，例如：

- **AP/WEB 分離規則**（如 LineBC）：WEB 層是否直接存取 DB → ERROR
- **分層限制**：Controller 是否繞過 Service 直接呼叫 DAO → ERROR
- **特定 Pattern 要求**：是否遵循專案要求的設計模式 → WARNING
- **回傳格式**：API 回傳是否使用標準 ApiResult → WARNING
- **例外處理慣例**：是否使用專案自訂 Exception → WARNING

#### 架構一致性檢查

對照「架構設計」中的類別清單：
- 預期的類別是否都已建立 → INFO
- 是否有預期外的類別 → INFO
- 介面方法是否與設計一致 → WARNING

### 6. 產出審查報告

格式：

```
## 📋 程式碼審查報告

**審查日期**：{日期}
**審查範圍**：{branch} ({N} 個 commit, {M} 個檔案)

### 統計
- 🔴 ERROR：{N} 項（必須修復）
- 🟡 WARNING：{N} 項（建議修復）
- 🔵 INFO：{N} 項（參考資訊）

### 🔴 ERROR

#### E1. {問題標題}
- **檔案**：`path/to/file.java:42`
- **問題**：{問題描述}
- **建議**：{修復建議}

### 🟡 WARNING

#### W1. {問題標題}
- **檔案**：`path/to/file.java:85`
- **問題**：{問題描述}
- **建議**：{修復建議}

### 🔵 INFO
（列表格式）

### ✅ 良好實踐
（值得肯定的程式碼設計）
```

### 7. 寫入 Notion

1. 使用 `notion-update-page` 的 `update_content`，將審查報告寫入「📝 開發日誌」區塊（帶時間戳）

2. 使用 `notion-update-page` 更新 Property：
   - 開發階段 = `程式碼審查`

### 8. 回傳結果

向使用者顯示：
- 審查報告摘要（ERROR / WARNING / INFO 數量）
- 重大問題清單（ERROR 項目）
- Notion 頁面連結
- 提示後續操作：
  ```
  若有 ERROR 項目，建議修復後重新執行 /feature-review。
  確認品質後，可使用 /feature-close 結案。
  ```

---

## 邊界情況

- **無 Git diff**：提示使用者先 commit 程式碼
- **架構設計區塊為空**：跳過架構一致性檢查，僅執行通用 + 專案特有檢查
- **專案無 CLAUDE.md**：僅執行通用檢查，提醒使用者建立 CLAUDE.md 以獲得更精確的審查
- **diff 過大（> 1000 行）**：聚焦 ERROR 和 WARNING，簡化 INFO 輸出
