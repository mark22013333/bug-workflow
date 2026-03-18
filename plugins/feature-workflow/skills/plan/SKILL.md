---
name: plan
description: 本地規劃（spec/db/arch/investigate/root-cause/fix），所有產出寫入 .spec/ 目錄，不呼叫 Notion API。當使用者提到「plan」、「規劃」、「設計」時觸發此 Skill。
---

# plan — 本地規劃（零 Notion 呼叫）

讀取 `.spec/{slug}/` 目錄的需求描述，啟動 Agent 進行規劃和設計，所有產出寫入本地 `.spec/` 檔案。**不呼叫任何 Notion API**，完全離線作業。

---

## 使用方式

```
/plan                      # Feature: 完整規劃（spec → db → arch）
/plan spec                 # Feature: 只做技術規格
/plan db                   # Feature: 只做 DB 設計
/plan arch                 # Feature: 只做架構設計
/plan investigate          # Bug: 記錄調查過程
/plan root-cause           # Bug: 根因分析
/plan fix                  # Bug: 修復方案
```

---

## 流程

> **前置檢查**：參照 bug-workflow plugin 的 `references/prerequisites.md` 檢查 CLAUDE.md 是否存在。

### 1. 定位活躍任務

按優先順序匹配：

1. 從 Git branch 匹配：讀取 `.spec/_index.md`，找「分支」欄位與當前 `git branch --show-current` 匹配的任務
2. 若匹配到的任務 status 為 `暫停` → 提示「此任務已暫停，是否恢復？[Y/n]」，確認後執行 unpark 流程（見 `/plan-status --unpark`）再繼續
3. 若只有一個進行中任務 → 自動選定
4. 若多個進行中 → 列出供選擇
5. 若無進行中 → 提示先執行 `/plan-start`

選定後讀取 `.spec/{slug}/README.md`，取得 `type`、`name`、`status`、`tech_stack` 等元資訊。

### 2. 解析子命令

| 子命令 | 適用類型 | 產出檔案 | 前置檔案 |
|--------|---------|---------|---------|
| （無） | feature | spec.md → db.md + db.sql → arch.md | README.md |
| `spec` | feature | spec.md | README.md |
| `db` | feature | db.md + db.sql | spec.md（建議但非必要） |
| `arch` | feature | arch.md | spec.md + db.md（建議但非必要） |
| `investigate` | bug | investigation.md | README.md |
| `root-cause` | bug | root-cause.md | investigation.md（建議） |
| `fix` | bug | fix.md | root-cause.md（建議） |

若子命令與類型不匹配（如 Feature 用 `investigate`），提示使用者。

### 3. 讀取專案上下文

以下資訊收集一次，供所有子命令使用（供所有子命令使用）：

#### 3-1. 專案 CLAUDE.md

讀取 `pwd` 下最近的 CLAUDE.md（向上搜尋），取得技術棧、架構模式、分層規則、命名慣例。

#### 3-2. 技術棧資訊

從 `.spec/{slug}/README.md` 的 `tech_stack` 欄位，對照設定檔中的技術棧定義。

#### 3-3. 現有程式碼參考

使用 Glob/Grep 掃描與需求相關的現有程式碼（1-2 個 Controller、Service、Entity），了解 API 風格、命名慣例。

#### 3-4. 規範檔案

讀取可用的規範檔案：
- `~/.claude/rules/database.md`（DB 設計時）
- `~/.claude/rules/design-patterns.md`（架構設計時）
- `~/.claude/rules/java-performance.md`（效能相關）

### 4. 執行規劃（依子命令分支）

---

#### 4-A. `spec` — 技術規格書

使用 **Agent tool** 啟動 subagent（model: opus），prompt 指示如下：

**輸入來源**：
- 需求描述從 `.spec/{slug}/README.md` 讀取（非 Notion）
- 若使用者在指令中補充需求，合併到輸入

**輸出目標**：
- 使用 Write tool 寫入 `.spec/{slug}/spec.md`（非 Notion）

**Agent 額外指示**：
```
產出完整技術規格，包含：
1. 功能範圍（In Scope / Out of Scope）
2. API 端點設計（表格格式）
3. 業務邏輯規則
4. 錯誤處理策略
5. 分層決策
6. 效能需求

在文件最後附加判斷區塊：
---
## 判斷
- FRONTEND_REQUIRED: true/false
- FRONTEND_TECH: JSP/Vue/React/無
- DB_REQUIRED: true/false
- DB_TABLES: 預估的表清單
```

完成後更新 README.md 的 `status: 規格設計`。

---

#### 4-B. `db` — 資料庫設計

使用 **Agent tool** 啟動 subagent（model: opus），prompt 指示如下：

**輸入來源**：
- 技術規格從 `.spec/{slug}/spec.md` 讀取
- 若 spec.md 不存在，提示建議先執行 `/plan spec`，但允許從 README.md 需求描述直接設計

**輸出目標**：
- 設計文件 → `.spec/{slug}/db.md`
- SQL 檔案 → `.spec/{slug}/db.sql`（含 CREATE TABLE / INDEX / 範例資料 / Rollback SQL）

完成後更新 README.md 的 `status: DB 設計`。

---

#### 4-C. `arch` — 架構設計

使用 **Agent tool** 啟動 subagent（model: opus），prompt 指示如下：

**輸入來源**：
- 技術規格從 `.spec/{slug}/spec.md` 讀取
- DB 設計從 `.spec/{slug}/db.md` 讀取
- 若檔案不存在，允許從 README.md 需求描述設計

**輸出目標**：
- 架構設計 → `.spec/{slug}/arch.md`（含 Mermaid 架構圖、類別清單、介面定義、設計模式）

完成後更新 README.md 的 `status: 架構設計`。

---

#### 4-D. `investigate` — Bug 調查記錄

不啟動 Agent，**由使用者互動填寫**：

1. 提示使用者提供調查資訊（或從指令參數讀取）：
   - 關鍵 Log
   - 相關 SQL 查詢
   - 初步判斷

2. 寫入 `.spec/{slug}/investigation.md`：

```markdown
# 調查過程

## 調查時間
{當前日期}

## 關鍵 Log
{使用者提供}

## 相關 SQL 查詢
{使用者提供}

## 初步判斷
{使用者提供}
```

若使用者在指令後附帶大段文字（如 log），直接寫入對應區塊。

完成後更新 README.md 的 `status: 調查中`。

---

#### 4-E. `root-cause` — Bug 根因分析

可由使用者填寫或由 Agent 輔助分析：

1. 若 `investigation.md` 存在，讀取調查過程
2. 結合專案程式碼（使用 Grep/Read 定位相關程式碼）
3. 寫入 `.spec/{slug}/root-cause.md`：

```markdown
# 根因分析

## 問題根因
{分析結果}

## 問題檔案
{檔案路徑清單}

## 問題程式碼
{相關程式碼片段}

## 影響範圍
{影響的功能/API/頁面}
```

完成後更新 README.md 的 `status: 根因確認`。

---

#### 4-F. `fix` — Bug 修復方案

使用 Agent 根據根因分析，設計修復方案：

**輸入來源**：
- `.spec/{slug}/root-cause.md`
- `.spec/{slug}/investigation.md`
- 相關程式碼

**輸出目標**：
- `.spec/{slug}/fix.md`：

```markdown
# 修復方案

## 修改檔案清單
| 檔案 | 層級 | 修改說明 |
|------|------|---------|

## 修改策略
{具體修改步驟}

## 修改後程式碼
{關鍵程式碼片段}

## 驗證方式
- [ ] {驗證項目 1}
- [ ] {驗證項目 2}

## 迴歸風險
{可能影響的其他功能}
```

完成後更新 README.md 的 `status: 修復方案`。

### 5. 完整規劃模式（無子命令）

若使用者執行 `/plan` 不帶子命令，依序執行：

1. **spec**（技術規格）
2. **db**（若 spec 判斷 DB_REQUIRED=true）
3. **arch**（架構設計）

每個階段完成後向使用者展示摘要，確認後再執行下一個。使用者可隨時中斷。

### 5.5 一致性驗證（自動）

每個子命令完成後（或完整規劃的全部子命令完成後），**自動**執行交叉比對，使用者無需手動觸發。

#### 檢查項目

| 檢查項目 | 比對來源 | 具體檢查 |
|---------|---------|---------|
| API-DB 一致性 | spec.md ↔ db.md | spec 中每個 API 的請求/回應欄位，db.md 是否有對應表欄位 |
| DB-Arch 一致性 | db.md ↔ arch.md | db.md 的每個表，arch.md 是否有對應 POJO 和 Mapper |
| API-Arch 一致性 | spec.md ↔ arch.md | spec 的每個 API 端點，arch.md 是否有對應 Controller 方法 |
| 判斷區塊完整性 | spec.md | FRONTEND_REQUIRED / DB_REQUIRED 是否為 true/false |
| DB_TABLES 對應 | spec.md ↔ db.md | 判斷區塊的 DB_TABLES 是否與 db.md 表清單一致 |

> **觸發條件**：僅在比對的兩端檔案都存在時才執行該項檢查。例如只執行 `/plan spec` 時，因 db.md 和 arch.md 不存在，僅檢查「判斷區塊完整性」。

#### 分類

- 🔴 **不一致**：文件間矛盾（如 spec 定義的 API 欄位在 db.md 找不到對應表欄位）
- 🟡 **遺漏**：缺少預期內容（如 db.md 有表但缺少索引建議）
- 🟢 **良好**：交叉比對通過

#### 自動修復流程

發現問題時：
1. 顯示「發現 N 個不一致，修復中...（第 1/2 輪）」
2. 備份受影響的文件（`{file}.bak`）
3. 直接修改受影響的文件，補齊遺漏或修正矛盾
4. 重新執行檢查（最多 **2 輪**）
5. 2 輪後仍有問題 → 列出剩餘摘要，**不阻擋流程**

#### 輸出範例

全部通過：
```
✅ 一致性驗證通過（3 項交叉比對）
```

有修復：
```
🔧 一致性驗證完成（修復 2 項）
  - 🔴 修復：arch.md 缺少 ExportController → 已補充
  - 🟡 修復：db.md 缺少 idx_push_date 索引 → 已補充
```

剩餘問題（2 輪後仍存在）：
```
⚠️ 一致性驗證完成（修復 1 項，剩餘 1 項需人工確認）
  - 🔴 修復：db.md 缺少 export_log 表 → 已補充
  - 🟡 剩餘：spec.md 的 /api/export 回應欄位 fileSize 型別與 db.md 不一致（BIGINT vs INT）
```

### 6. 更新日誌

每次規劃完成，在 `.spec/{slug}/log.md` 追加一筆紀錄：

```markdown
### [{日期}] {子命令名稱}完成
- 產出檔案：{檔案路徑}
- 摘要：{一句話描述}
```

若 `log.md` 不存在則建立。

### 7. 回傳結果

```
{子命令} 規劃完成！

📁 產出檔案：.spec/{slug}/{檔案名稱}
📊 狀態：{新狀態}

後續可使用：
  • /plan {下一步建議}  — {說明}
  • /plan-build         — Agent Teams 產生程式碼
  • /plan-status        — 查看任務狀態
```

---

## Gotchas

- **spec.md「判斷」區塊格式是 plan-build 的入口**：`FRONTEND_REQUIRED` 和 `DB_REQUIRED` 的值直接決定 plan-build 的團隊組成（是否啟動 Frontend Agent、是否產生 DB 相關程式碼）。格式錯誤（如用中文「是/否」而非 `true/false`）會 fallback 到預設值，可能導致團隊組成不符預期。
- **Agent subagent 的 model 參數**：prompt 中寫「使用 Opus 模型」只是自然語言指示，不保證生效。必須在 Agent tool 的 `model` 參數實際設定 `"opus"`，才能確保使用指定模型。
- **重新執行覆蓋已有檔案**：覆蓋前會備份到 `{file}.bak`，但 `.bak` 只保留一份。第二次重新執行時，前一份 `.bak` 也會被覆蓋，無法回溯到更早的版本。

---

## 邊界情況

- **`.spec/` 目錄不存在**：提示先執行 `/plan-start`
- **找不到活躍任務**：提示先執行 `/plan-start` 或檢查 `_index.md`
- **前置檔案不存在**：提示建議先執行前置步驟，但不強制阻擋
- **使用者想修改已有的規劃檔案**：重新執行同一子命令會覆蓋（先備份舊版到 `{file}.bak`）
- **Bug 類型執行 Feature 子命令**：提示不適用，建議正確的子命令
