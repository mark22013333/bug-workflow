---
name: feature-arch
description: 使用 Agent(opus) 根據技術規格與 DB 設計，產出分層架構圖（Mermaid）、類別清單、介面定義、設計模式選擇，寫入 Notion 頁面。當使用者提到「feature-arch」、「架構設計」、「設計架構」、「分層設計」時觸發此 Skill。
---

# feature-arch — 架構設計

使用 Agent（Opus 模型）根據技術規格與 DB 設計，產出分層架構設計，並寫入 Notion 功能頁面的「🏗️ 架構設計」區塊。

---

## 設定檔

執行前依序檢查以下路徑，讀取第一個找到的設定檔：
1. `~/.claude-company/feature-workflow-config.md`（公司環境）
2. `~/.claude/feature-workflow-config.md`（個人環境）

若都不存在，提示使用者先執行 `/feature-setup`。

---

## 前置條件

- 已使用 `/feature-start` 建立功能條目
- 建議已執行 `/feature-spec` 和 `/feature-db`（非強制）

---

## 流程

### 1. 定位目標功能 Notion 頁面

使用與 feature-update 相同的定位邏輯。

### 2. 取得設計資訊

使用 `notion-fetch` 取得目標頁面完整內容，擷取：
- 「📐 技術規格」區塊（API 設計、業務邏輯）
- 「🗄️ 資料庫設計」區塊（表結構）

若使用者在指令中提供了補充說明，合併到設計輸入中。

### 3. 讀取專案上下文

收集以下資訊作為 Agent 的輸入：

#### 3-1. 專案 CLAUDE.md

讀取 CLAUDE.md，識別架構模式、分層規則、設計模式慣例。

#### 3-2. 專案 package 結構

使用 Glob 掃描專案的 package 結構，識別：
- Controller 層位置和慣例
- Service 層位置（是否有 Interface + Impl？）
- DAO/Repository 層位置
- Model/Entity/POJO/DTO 位置
- 是否有 Converter/Mapper 層
- 工具類位置

#### 3-3. 現有功能呼叫鏈

讀取 1-2 個現有功能的完整呼叫鏈（Controller → Service → DAO），理解：
- 方法命名慣例
- 依賴注入方式
- 事務管理方式

#### 3-4. 設計模式參考

讀取 `~/.claude/rules/design-patterns.md`（若存在）。

### 4. 啟動架構設計 Agent

使用 **Agent tool** 啟動 subagent，參數：
- **model**: `opus`
- **prompt**: 組合以下內容傳入 Agent：

```
你是一位資深後端架構設計師。

## 專案上下文
{專案 CLAUDE.md 內容}

## 專案 package 結構
{掃描到的 package 結構}

## 現有功能呼叫鏈範例
{現有 Controller → Service → DAO 程式碼片段}

## 設計模式參考
{~/.claude/rules/design-patterns.md 內容，若存在}

## 技術規格
{從 Notion 取得的技術規格}

## 資料庫設計
{從 Notion 取得的 DB 設計}

## 補充說明
{使用者額外提供的補充}

## 任務
根據以上資訊，設計分層架構，包含：
1. 分層架構圖（Mermaid 語法，展示各層之間的呼叫關係）
2. 類別/介面清單（表格格式，含完整 package 路徑和說明）
3. 每個 Service 介面的方法定義（method signature + Javadoc）
4. 設計模式選擇與理由（表格格式：模式、應用位置、理由）
5. 層間依賴關係說明

注意：
- 遵循該專案已有的分層模式（不強加外部模式）
- 類別清單必須含完整 package 路徑（根據專案結構推斷）
- 設計模式建議要附理由，並與專案現有模式保持一致
輸出使用繁體中文，Markdown 格式。
```

### 5. 寫入 Notion

Agent 完成後：

1. 使用 `notion-update-page` 的 `update_content`，將架構設計內容寫入「🏗️ 架構設計」區塊

2. 使用 `notion-update-page` 更新 Property：
   - 開發階段 = `架構設計`

### 6. 回傳結果

向使用者顯示：
- 設計摘要（類別數量、設計模式）
- Mermaid 架構圖（直接顯示）
- Notion 頁面連結
- 提示後續指令：
  ```
  架構設計已寫入 Notion！後續可使用：
  • /feature-scaffold [--dry-run] — 根據架構產生程式碼骨架
  • /feature-update               — 調整架構設計
  ```

---

## 邊界情況

- **技術規格和 DB 設計都為空**：提示建議先執行前置步驟，但允許使用者口頭描述
- **專案 package 結構特殊**：盡可能推斷，無法推斷的詢問使用者
- **設計模式不適用**：可以不建議設計模式，簡單功能不需要過度設計
- **使用者想修改架構**：可再次執行 `/feature-arch` 或用 `/feature-update 架構調整：...` 局部更新
