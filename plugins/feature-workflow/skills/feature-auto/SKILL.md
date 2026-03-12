---
name: feature-auto
description: 讀取本地規格書檔案，自動依序執行 feature-start → spec → db → arch → scaffold 完成功能建立。當使用者提到「feature-auto」、「自動開發」、「讀取規格書」、「匯入規格」時觸發此 Skill。
---

# feature-auto — 規格書驅動的自動化功能建立

讀取本地規格書檔案（Markdown / 純文字），自動依序執行 feature workflow 的各階段，從建立 Notion 條目到產生程式碼骨架，一氣呵成。

---

## 設定檔

執行前依序檢查以下路徑，讀取第一個找到的設定檔：
1. `~/.claude-company/feature-workflow-config.md`（公司環境）
2. `~/.claude/feature-workflow-config.md`（個人環境）

若都不存在，提示使用者先執行 `/feature-setup`。

---

## 使用方式

```
/feature-auto <規格書檔案路徑> [選項]
```

### 參數

| 參數 | 說明 |
|------|------|
| `<規格書檔案路徑>` | 必填，本地檔案的相對或絕對路徑 |
| `--stop-at <階段>` | 執行到指定階段後暫停（見下方階段列表） |
| `--dry-run` | scaffold 階段僅預覽，不建立檔案 |
| `--skip <階段,階段>` | 跳過指定階段（逗號分隔） |

### 階段列表

| 階段 ID | 對應 Skill | 說明 |
|---------|-----------|------|
| `start` | `/feature-start` | 建立 Notion 條目 + Git branch |
| `spec` | `/feature-spec` | 技術規格書 |
| `db` | `/feature-db` | 資料庫設計 |
| `arch` | `/feature-arch` | 架構設計 |
| `scaffold` | `/feature-scaffold` | 程式碼骨架 |

### 範例

```bash
# 完整流程
/feature-auto doc/訂閱推播統計.md

# 只到規格書階段
/feature-auto doc/訂閱推播統計.md --stop-at spec

# 跳過 DB 設計（功能不涉及資料庫）
/feature-auto doc/訂閱推播統計.md --skip db

# scaffold 階段僅預覽
/feature-auto doc/訂閱推播統計.md --dry-run
```

---

## 規格書格式

規格書無強制格式，但建議包含以下資訊以獲得最佳效果：

```markdown
# 功能名稱

## 需求描述
- 功能目的
- 目標使用者
- 使用情境

## 驗收條件
- [ ] 條件 1
- [ ] 條件 2

## 技術備註（選填）
- API 路徑偏好
- DB 設計偏好
- 效能需求
- 其他限制
```

若規格書內容簡略，各階段的 Agent 會根據專案 CLAUDE.md 自動補充推斷。

---

## 流程

### 1. 讀取規格書

使用 Read tool 讀取使用者指定的檔案路徑。

若檔案不存在或無法讀取，顯示錯誤訊息並終止。

從規格書中解析：
- **功能名稱**：取第一個 `#` 標題，若無則取檔名（去掉副檔名）
- **需求描述**：取全文或「需求描述」區塊
- **技術備註**：取「技術備註」區塊（若有）
- **驗收條件**：取「驗收條件」區塊（若有）

### 2. 確認執行計畫

向使用者展示執行計畫，等待確認：

```
📄 規格書：doc/訂閱推播統計.md
📝 功能名稱：訂閱推播統計報表
📋 需求摘要：{前 3 行}

即將依序執行：
  1. ✅ feature-start  — 建立 Notion 條目 + Git branch
  2. ✅ feature-spec   — 技術規格書（Agent）
  3. ✅ feature-db     — 資料庫設計（Agent）
  4. ✅ feature-arch   — 架構設計（Agent）
  5. ✅ feature-scaffold — 程式碼骨架（Agent）

預估耗時：5-10 分鐘（依規格複雜度而定）

確認執行？[Y/n]
```

若使用者有指定 `--stop-at` 或 `--skip`，相應調整計畫展示（跳過的階段標示 ⏭️，暫停後的階段標示 ⏸️）。

### 3. 執行 feature-start

使用 Skill tool 觸發 `/feature-start`，傳入功能名稱。

**傳入方式**：將規格書中的功能名稱作為參數

**互動處理**：
- 優先順序：若規格書中未指定，使用預設「中」
- 難度：若規格書中未指定，根據需求描述的複雜度自動判斷
- Git branch：自動建立（從功能名稱產生英文簡寫）

**取得結果**：記錄 Notion 頁面 URL，供後續階段使用。

### 4. 執行 feature-spec

使用 Skill tool 觸發 `/feature-spec`，將規格書完整內容作為補充需求傳入。

**傳入方式**：`/feature-spec {規格書完整內容}`

Agent 會結合規格書內容和專案 CLAUDE.md，產出完整技術規格書並寫入 Notion。

**階段完成提示**：

```
✅ [1/5] feature-start 完成 — Notion 頁面已建立
✅ [2/5] feature-spec 完成 — 技術規格書已寫入
⏳ [3/5] feature-db 執行中...
```

### 5. 執行 feature-db

使用 Skill tool 觸發 `/feature-db`，將規格書中的技術備註作為補充說明。

**傳入方式**：`/feature-db {技術備註中的 DB 相關內容}`

**SQL 檔案輸出**：自動選擇輸出到 `doc/{功能名稱}.sql`（不再互動詢問）。

### 6. 執行 feature-arch

使用 Skill tool 觸發 `/feature-arch`，將規格書中的技術備註作為補充說明。

**傳入方式**：`/feature-arch {技術備註中的架構相關內容}`

### 7. 執行 feature-scaffold

使用 Skill tool 觸發 `/feature-scaffold`。

**dry-run 處理**：
- 若使用者指定 `--dry-run`，觸發 `/feature-scaffold --dry-run`，到此結束
- 否則觸發 `/feature-scaffold`，展示檔案清單後自動確認建立

### 8. 回傳總結報告

所有階段完成後，向使用者展示完整報告：

```
🎉 功能建立完成！

📄 規格書：doc/訂閱推播統計.md
📝 功能名稱：訂閱推播統計報表
🔀 Git branch：feature/subscription-push-statistics
📊 Notion 頁面：https://notion.so/xxx

已完成階段：
  ✅ feature-start   — Notion 條目 + Git branch
  ✅ feature-spec    — 3 個 API、5 項業務規則
  ✅ feature-db      — 2 個表、4 個索引 → doc/訂閱推播統計.sql
  ✅ feature-arch    — 8 個類別、策略模式
  ✅ feature-scaffold — 8 個檔案已建立

後續操作：
  1. 實作 Service 中的 TODO 業務邏輯
  2. 隨時用 /feature-update 記錄進度
  3. 完成後用 /feature-review 檢查品質
  4. 最後用 /feature-close 結案
```

---

## 錯誤處理

### 單一階段失敗

若某個階段執行失敗（如 Notion API 錯誤、Agent 逾時），不自動終止整個流程：

```
❌ [3/5] feature-db 失敗：Notion API 回應 429 (Rate Limit)

選擇：
1. 重試此階段
2. 跳過此階段，繼續下一步
3. 終止流程（已完成的階段不受影響）
```

### 中途終止後恢復

若使用者中途終止或階段失敗後選擇終止，已完成的階段不受影響。使用者可手動執行剩餘的 skill 繼續：

```
已完成 start + spec，若要繼續：
  /feature-db       ← 從 DB 設計繼續
  /feature-arch     ← 或跳過 DB，直接設計架構
```

---

## 邊界情況

- **規格書檔案不存在**：顯示錯誤訊息，提示檢查路徑
- **規格書為空或內容過短（< 10 字）**：警告使用者內容過少可能影響 Agent 產出品質，詢問是否繼續
- **規格書過長（> 500 行）**：正常處理，Agent 會自行擷取重點
- **設定檔不存在**：提示使用者先執行 `/feature-setup`
- **非 Git repo**：跳過 branch 建立，其餘流程正常執行
- **使用者在確認時取消**：直接終止，不執行任何階段
