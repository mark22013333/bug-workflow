---
name: feature-auto
description: 讀取本地規格書檔案，以 Leader 委派模式自動依序執行 feature-start → spec → db → arch → scaffold 完成功能建立。當使用者提到「feature-auto」、「自動開發」、「讀取規格書」、「匯入規格」時觸發此 Skill。
---

# feature-auto — 規格書驅動的 Leader 委派自動化功能建立

讀取本地規格書檔案（Markdown / 純文字），以 **Leader 委派模式**自動依序執行 feature workflow 的各階段，從建立 Notion 條目到產生程式碼骨架，一氣呵成。

---

## 前置條件

### 環境變數

必須啟用 Agent Teams 實驗功能：

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

可加入 shell profile（`~/.zshrc` 或 `~/.bashrc`）使其永久生效。

### Claude Code 設定

建議啟用 Extended Thinking，讓 Agent 在複雜推理時有更充分的思考空間：

在 `~/.claude/settings.json` 中加入：

```json
{
  "alwaysThinkingEnabled": true
}
```

或在 Claude Code 對話中執行 `/config` 切換。

### 設定檔

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

## Leader 委派架構

本 Skill 採用 **Leader 委派模式**。Leader（你自己）是唯一的協調者，負責：
1. 讀取規格書並建立 Notion 條目
2. 依序啟動每個 Teammate（Subagent）
3. **收集每個 Teammate 的完整產出**
4. **將累積的上下文直接注入下一個 Teammate 的 prompt**（不依賴 Notion 讀取）
5. 彙整所有結果，產出總結報告

### 資訊流

```
Leader
  │
  │  ① 讀取規格書 + 執行 feature-start
  │     保存：{規格書內容}、{Notion 頁面 URL}
  │
  ├──► spec-analyst
  │     輸入：規格書內容
  │     輸出：技術規格書（寫入 Notion + 完整內容回傳 Leader）
  │     Leader 保存：{spec_output}
  │
  ├──► db-designer
  │     輸入：規格書 + spec_output（Leader 直接傳入）
  │     輸出：DB 設計（寫入 Notion + 完整內容回傳 Leader）
  │     Leader 保存：{db_output}
  │
  ├──► arch-designer
  │     輸入：規格書 + spec_output + db_output（Leader 直接傳入）
  │     輸出：架構設計（寫入 Notion + 完整內容回傳 Leader）
  │     Leader 保存：{arch_output}
  │
  └──► code-generator
        輸入：spec_output + db_output + arch_output（Leader 直接傳入）
        輸出：程式碼骨架（寫入檔案 + Notion + 檔案清單回傳 Leader）
```

### 與舊模式的差異

| | 舊模式（Notion 中繼） | 新模式（Leader 委派） |
|--|---------------------|-------------------|
| **下游 Agent 如何取得上游產出** | 自己去 Notion fetch | Leader 直接注入 prompt |
| **Notion 的角色** | 中繼站（讀 + 寫） | 持久化儲存（僅寫入） |
| **資訊完整度** | 取決於 Agent 解析 Notion 的能力 | Leader 控制，零損失 |
| **Notion API 依賴** | 每階段 2 次（讀 + 寫） | 每階段 1 次（僅寫入） |
| **失敗容錯** | Notion 讀取失敗 = 下游斷鏈 | 不受 Notion 讀取影響 |

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
🤖 執行模式：Leader 委派（Leader + 4 Teammates）

即將依序執行：
  1. ✅ feature-start   — Leader 建立 Notion 條目 + Git branch
  2. ✅ feature-spec    — spec-analyst 產出技術規格書
  3. ✅ feature-db      — db-designer 設計資料庫
  4. ✅ feature-arch    — arch-designer 設計架構
  5. ✅ feature-scaffold — code-generator 產生程式碼骨架

確認執行？[Y/n]
```

若使用者有指定 `--stop-at` 或 `--skip`，相應調整計畫展示（跳過的階段標示 ⏭️，暫停後的階段標示 ⏸️）。

### 3. Leader 執行 feature-start

Leader 直接執行此階段（不需 Teammate），使用 Skill tool 觸發 `/feature-start`：

- 將規格書中的功能名稱作為參數
- 優先順序：若規格書中未指定，使用預設「中」
- 難度：若規格書中未指定，根據需求描述的複雜度自動判斷
- Git branch：自動建立（從功能名稱產生英文簡寫）
- **記錄 Notion 頁面 URL**，供後續 Teammates 寫入使用

### 4. 建立 Agent Team 並依序委派

使用 **TeamCreate** 建立團隊。根據使用者指定的選項（`--skip`、`--stop-at`），僅建立需要的 Teammates。

每個 Teammate 使用 **Agent tool** 啟動，指定 `model: opus`。

**Leader 在每個 Teammate 完成後，必須保存其完整產出內容，並在啟動下一個 Teammate 時注入 prompt。**

---

#### spec-analyst（技術規格分析師）

**啟動時機**：feature-start 完成後

**Leader 注入的上下文**：規格書原始內容

**Prompt**：

```
你是一位資深技術規格分析師，負責此次功能開發的技術規格書產出。

## 你的任務
結合規格書需求與專案上下文，產出完整技術規格書，寫入 Notion 並將完整內容回傳。

## 規格書原始需求
{規格書完整內容}

## Notion 頁面 URL
{Notion 頁面 URL}

## 執行步驟
1. 讀取設定檔（~/.claude-company/feature-workflow-config.md 或 ~/.claude/feature-workflow-config.md）
2. 讀取當前專案的 CLAUDE.md，了解技術棧和架構慣例
3. 使用 Glob/Grep 掃描專案中相關的 Controller 和 Service（1-2 個），了解 API 風格
4. 產出技術規格書，包含：
   - 功能範圍（In Scope / Out of Scope）
   - API 端點設計（表格：端點、Method、請求參數/Body、回傳格式、說明）
   - 業務邏輯規則（每個 API 的核心邏輯、前置驗證、處理流程、邊界條件）
   - 錯誤處理策略（表格：錯誤場景、處理方式、HTTP 狀態碼）
   - 分層決策（哪些邏輯放哪一層）
   - 效能需求（預期資料量、分頁、快取）
5. 使用 notion-update-page 將規格寫入「📐 技術規格」區塊
6. 使用 notion-update-page 更新開發階段 = 「規格設計」

API 路徑、回傳格式、錯誤處理必須遵循專案既有慣例。
輸出使用繁體中文。

## 重要：回傳格式
完成後，你必須回傳以下兩部分：
1. **摘要**：產出了幾個 API、幾項業務規則（一行）
2. **完整技術規格內容**：你寫入 Notion 的完整 Markdown 內容（用 ```markdown 包裹）

Leader 需要你的完整產出來傳遞給下一階段的 Teammate。
```

**Leader 收到回傳後**：
- 解析摘要 → 顯示進度
- 保存完整技術規格內容為 `{spec_output}`

---

#### db-designer（資料庫設計師）

**啟動時機**：spec-analyst 完成後

**Leader 注入的上下文**：規格書原始內容 + spec_output

**Prompt**：

```
你是一位資深資料庫設計師，負責此次功能開發的資料庫設計。

## 你的任務
根據技術規格設計資料表結構、索引和遷移 SQL，寫入 Notion 並將完整內容回傳。

## 規格書原始需求（參考）
{規格書完整內容}

## 前一階段產出：技術規格書（由 Leader 提供，無需從 Notion 讀取）
{spec_output}

## Notion 頁面 URL
{Notion 頁面 URL}

## 執行步驟
1. 讀取設定檔（~/.claude-company/feature-workflow-config.md 或 ~/.claude/feature-workflow-config.md）
2. 讀取當前專案的 CLAUDE.md，識別 DB 類型（MSSQL / MySQL / PostgreSQL）
3. 使用 Glob 掃描現有 Entity/POJO（2-3 個範本），了解命名慣例、公共欄位、註解風格
4. 讀取 ~/.claude/rules/database.md（若存在），取得資料庫規範
5. 根據上方的技術規格書，設計資料庫結構，包含：
   - CREATE TABLE（正確的 DB 語法，含欄位註解和公共欄位）
   - CREATE INDEX（遵循專案命名慣例）
   - 範例資料 INSERT（3-5 筆）
   - Rollback SQL（DROP TABLE / DROP INDEX）
6. 使用 notion-update-page 將 DB 設計寫入「🗄️ 資料庫設計」區塊
7. 使用 notion-update-page 更新開發階段 = 「DB 設計」
8. 使用 Write tool 將 SQL 輸出到 doc/{功能名稱}.sql

命名慣例、資料型別、索引風格必須與現有表一致。
輸出使用繁體中文。

## 重要：回傳格式
完成後，你必須回傳以下兩部分：
1. **摘要**：新增幾個表、幾個索引、SQL 檔案路徑（一行）
2. **完整 DB 設計內容**：你寫入 Notion 的完整 Markdown 內容（用 ```markdown 包裹）

Leader 需要你的完整產出來傳遞給下一階段的 Teammate。
```

**Leader 收到回傳後**：
- 解析摘要 → 顯示進度
- 保存完整 DB 設計內容為 `{db_output}`

---

#### arch-designer（架構設計師）

**啟動時機**：db-designer 完成後

**Leader 注入的上下文**：規格書原始內容 + spec_output + db_output

**Prompt**：

```
你是一位資深後端架構設計師，負責此次功能開發的分層架構設計。

## 你的任務
根據技術規格與 DB 設計，產出分層架構設計，寫入 Notion 並將完整內容回傳。

## 規格書原始需求（參考）
{規格書完整內容}

## 前一階段產出：技術規格書（由 Leader 提供，無需從 Notion 讀取）
{spec_output}

## 前一階段產出：資料庫設計（由 Leader 提供，無需從 Notion 讀取）
{db_output}

## Notion 頁面 URL
{Notion 頁面 URL}

## 執行步驟
1. 讀取設定檔（~/.claude-company/feature-workflow-config.md 或 ~/.claude/feature-workflow-config.md）
2. 讀取當前專案的 CLAUDE.md，識別架構模式和分層規則
3. 使用 Glob 掃描專案 package 結構，識別各層位置和慣例
4. 讀取 1-2 個現有功能的完整呼叫鏈（Controller → Service → DAO）
5. 讀取 ~/.claude/rules/design-patterns.md（若存在）
6. 根據上方的技術規格和 DB 設計，產出架構設計，包含：
   - 分層架構圖（Mermaid 語法）
   - 類別/介面清單（表格，含完整 package 路徑和說明）
   - 每個 Service 介面的方法定義（method signature + Javadoc）
   - 設計模式選擇與理由
   - 層間依賴關係說明
7. 使用 notion-update-page 將架構設計寫入「🏗️ 架構設計」區塊
8. 使用 notion-update-page 更新開發階段 = 「架構設計」

遵循專案已有的分層模式，不強加外部模式。
輸出使用繁體中文。

## 重要：回傳格式
完成後，你必須回傳以下兩部分：
1. **摘要**：設計了幾個類別、使用了哪些設計模式（一行）
2. **完整架構設計內容**：你寫入 Notion 的完整 Markdown 內容（用 ```markdown 包裹）

Leader 需要你的完整產出來傳遞給下一階段的 Teammate。
```

**Leader 收到回傳後**：
- 解析摘要 → 顯示進度
- 保存完整架構設計內容為 `{arch_output}`

---

#### code-generator（程式碼產生器）

**啟動時機**：arch-designer 完成後

**Leader 注入的上下文**：spec_output + db_output + arch_output（不需要規格書原文，前三階段已涵蓋）

**Prompt**：

```
你是一位嚴謹的程式碼產生器。最重要的原則：產出的程式碼風格必須與專案現有程式碼完全一致。

## 你的任務
根據前面階段的設計產出，按專案既有風格產生程式碼骨架。

## 前面階段產出：技術規格書（由 Leader 提供，無需從 Notion 讀取）
{spec_output}

## 前面階段產出：資料庫設計（由 Leader 提供，無需從 Notion 讀取）
{db_output}

## 前面階段產出：架構設計（由 Leader 提供，無需從 Notion 讀取）
{arch_output}

## Notion 頁面 URL
{Notion 頁面 URL}

## 執行步驟
1. 讀取設定檔（~/.claude-company/feature-workflow-config.md 或 ~/.claude/feature-workflow-config.md）
2. 從設定檔的「專案對應」表取得當前專案的技術棧 ID
3. 根據技術棧 ID 掃描專案中同類型的現有程式碼各一個作為風格範本：
   - POJO/Entity、Mapper/Repository、Mapper XML（若適用）、Service、Controller
4. 讀取每個範本的完整內容，學習 package、import 順序、註解風格、縮排
5. 根據上方的架構設計（類別清單、介面定義）和 DB 設計（CREATE TABLE），產生所有程式碼檔案，要求：
   - package、import 順序、註解風格、縮排與範本完全一致
   - Getter/Setter 風格從範本學習（Lombok? 手動?）
   - Service 方法骨架含 TODO 註解標記待實作的業務邏輯
   - 註解和說明使用繁體中文
6. 先展示檔案清單表格（#、檔案路徑、類型、說明）{dry_run_instruction}
7. 使用 Write tool 依序建立檔案（先 Entity → Mapper → Service → Controller）
8. 使用 notion-update-page 將檔案清單寫入「📁 程式碼清單」區塊
9. 使用 notion-update-page 更新開發階段 = 「開發中」

完成後回報：建立了幾個檔案、檔案清單。
```

> `{dry_run_instruction}`：若使用者指定 `--dry-run`，替換為「展示檔案清單後停止，不建立檔案」；否則為空。

---

### 5. 依序委派並傳遞上下文

Leader 的核心職責是**收集完整產出並向下傳遞**。流程如下：

```
1. Leader 執行 feature-start（直接執行）
   → 保存 Notion 頁面 URL

2. 啟動 spec-analyst（注入：規格書內容）
   → 等待完成
   → 保存完整技術規格內容為 spec_output
   → 向使用者回報進度

3. 啟動 db-designer（注入：規格書 + spec_output）
   → 等待完成
   → 保存完整 DB 設計內容為 db_output
   → 向使用者回報進度

4. 啟動 arch-designer（注入：規格書 + spec_output + db_output）
   → 等待完成
   → 保存完整架構設計內容為 arch_output
   → 向使用者回報進度

5. 啟動 code-generator（注入：spec_output + db_output + arch_output）
   → 等待完成
   → 向使用者回報進度
```

**關鍵原則**：
- 每個 Teammate **不需要從 Notion 讀取上游產出**，Leader 已直接注入
- 每個 Teammate **仍然寫入 Notion**（持久化，供人閱讀和後續 skill 使用）
- Leader **保存完整產出**而非僅保存摘要，確保下游 Agent 拿到完整上下文

每個階段完成後，Leader 向使用者顯示進度：

```
✅ [1/5] feature-start 完成 — Notion 頁面已建立
✅ [2/5] spec-analyst 完成 — 3 個 API、5 項業務規則
✅ [3/5] db-designer 完成 — 2 個表、4 個索引 → doc/訂閱推播統計.sql
✅ [4/5] arch-designer 完成 — 8 個類別、策略模式
✅ [5/5] code-generator 完成 — 8 個檔案已建立
```

### 6. 回傳總結報告

所有階段完成後，向使用者展示完整報告：

```
🎉 功能建立完成！

📄 規格書：doc/訂閱推播統計.md
📝 功能名稱：訂閱推播統計報表
🔀 Git branch：feature/subscription-push-statistics
📊 Notion 頁面：https://notion.so/xxx
🤖 執行模式：Leader 委派（Leader + 4 Teammates）

已完成階段：
  ✅ feature-start    — Notion 條目 + Git branch
  ✅ spec-analyst     — 3 個 API、5 項業務規則
  ✅ db-designer      — 2 個表、4 個索引 → doc/訂閱推播統計.sql
  ✅ arch-designer    — 8 個類別、策略模式
  ✅ code-generator   — 8 個檔案已建立

後續操作：
  1. 實作 Service 中的 TODO 業務邏輯
  2. 隨時用 /feature-update 記錄進度
  3. 完成後用 /feature-review 檢查品質
  4. 最後用 /feature-close 結案
```

---

## 錯誤處理

### 單一階段失敗

若某個 Teammate 執行失敗（如 Notion API 錯誤、Agent 逾時），Leader 不自動終止整個流程：

```
❌ [3/5] db-designer 失敗：Notion API 回應 429 (Rate Limit)

選擇：
1. 重試此階段（重新啟動 Teammate，注入相同上下文）
2. 跳過此階段，繼續下一步（下游 Agent 的 prompt 中標記「DB 設計已跳過」）
3. 終止流程（已完成的階段不受影響）
```

**跳過階段時的上下文處理**：
- 若跳過 db-designer，Leader 在 arch-designer 的 prompt 中標記「DB 設計已跳過，請根據技術規格自行推斷所需的資料結構」
- `{db_output}` 設為空字串，不影響下游 prompt 結構

### Agent Teams 未啟用

若環境變數 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` 未設定或不為 `1`，顯示提示：

```
⚠️ Agent Teams 功能未啟用。請設定環境變數後重啟 Claude Code：

export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1

或加入 ~/.zshrc 使其永久生效。
```

### 中途終止後恢復

若使用者中途終止或階段失敗後選擇終止，已完成的階段不受影響（已寫入 Notion）。使用者可手動執行剩餘的 skill 繼續：

```
已完成 start + spec，若要繼續：
  /feature-db       ← 從 DB 設計繼續（會從 Notion 讀取 spec 產出）
  /feature-arch     ← 或跳過 DB，直接設計架構
```

> 注意：手動執行個別 skill 時，仍會從 Notion 讀取上游產出（個別 skill 不使用 Leader 委派模式）。

---

## 邊界情況

- **規格書檔案不存在**：顯示錯誤訊息，提示檢查路徑
- **規格書為空或內容過短（< 10 字）**：警告使用者內容過少可能影響 Agent 產出品質，詢問是否繼續
- **規格書過長（> 500 行）**：正常處理，Agent 會自行擷取重點
- **設定檔不存在**：提示使用者先執行 `/feature-setup`
- **非 Git repo**：跳過 branch 建立，其餘流程正常執行
- **使用者在確認時取消**：直接終止，不執行任何階段
- **Agent Teams 未啟用**：顯示環境變數設定提示
- **Teammate 回傳內容過大**：Leader 保存完整內容，但若累積上下文超過 context window 限制，對最早的階段產出進行摘要壓縮（優先壓縮規格書原文，因為 spec_output 已涵蓋其重點）
