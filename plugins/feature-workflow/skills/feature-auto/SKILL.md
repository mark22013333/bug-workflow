---
name: feature-auto
description: 讀取本地規格書檔案，以 Agent Teams 模式自動完成功能開發全流程（規格 → 設計 → 程式碼 → 審查）。支援並行開發、交叉審查、tmux 分割視窗。當使用者提到「feature-auto」、「自動開發」、「讀取規格書」、「匯入規格」時觸發此 Skill。
---

# feature-auto — Agent Teams 驅動的全流程自動化功能開發

讀取本地規格書檔案，以 **Agent Teams** 模式驅動 6 個階段的全流程自動化：規格分析 → 設計（並行）→ 程式碼產生（並行）→ 品質審查（並行），支援 Teammate 間 Mailbox 直接通訊與交叉審查。

---

## 前置條件

### 環境變數

必須啟用 Agent Teams 實驗功能：

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

可加入 shell profile（`~/.zshrc` 或 `~/.bashrc`）使其永久生效。

### Claude Code 設定

建議在 `~/.claude/settings.json` 中加入：

```json
{
  "alwaysThinkingEnabled": true,
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

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
| `--stop-at <階段>` | 執行到指定階段後暫停 |
| `--dry-run` | code 階段僅預覽，不建立檔案 |
| `--skip <階段,階段>` | 跳過指定階段（逗號分隔） |
| `--backend-only` | 強制跳過前端 Coder（即使 spec 判斷需要前端） |
| `--no-review` | 跳過 Code Review 階段 |

### 階段列表

| 階段 ID | 說明 | 模式 |
|---------|------|------|
| `start` | 建立 Notion 條目 + Git branch | Leader 直接執行 |
| `spec` | 技術規格書 | 單一 Teammate |
| `design` | DB 設計 + 架構設計 | **Design Team（並行 + 交叉審查）** |
| `code` | 程式碼骨架產生 | **Code Gen Team（後端 + 前端並行）** |
| `review` | 程式碼品質審查 | **Review Team（三位審查員並行）** |

### 範例

```bash
# 完整流程（6 階段全部執行）
/feature-auto doc/訂閱推播統計.md

# 只到設計階段
/feature-auto doc/訂閱推播統計.md --stop-at design

# 跳過 DB 設計（功能不涉及資料庫）
/feature-auto doc/訂閱推播統計.md --skip db

# 只產後端程式碼，不做 Review
/feature-auto doc/訂閱推播統計.md --backend-only --no-review

# Code 階段僅預覽
/feature-auto doc/訂閱推播統計.md --dry-run
```

---

## 規格書格式

規格書無強制格式，但建議包含以下資訊：

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
- 是否需要前端頁面
- DB 設計偏好
- 效能需求
```

若規格書內容簡略，各階段的 Agent 會根據專案 CLAUDE.md 自動補充推斷。

---

## 整體架構

```
Leader（delegate mode — 只協調不寫 code）
  │
  │  Phase 1: feature-start（Leader 直接執行）
  │
  ├─► Phase 2: spec-analyst（單一 Teammate）
  │     產出：技術規格書 + 判斷「是否需要前端」「是否需要 DB」
  │
  │  Phase 3: Design Team ─────────────────────────────────
  │  │                                                      │
  │  ├─► db-designer  ──┐                                   │
  │  └─► arch-designer ─┤ 並行工作                          │
  │                      ├─ 完成後互相 Review（Mailbox）      │
  │                      └─ 修正後回報 Leader                │
  │  ────────────────────────────────────────────────────────
  │
  │  Phase 4: Code Gen Team ────────────────────────────────
  │  │                                                      │
  │  ├─► backend-coder  ──┐                                 │
  │  └─► frontend-coder ──┤ 並行（各用 WorkTree）            │
  │       (條件性)         ├─ 完成後確認 API 契約（Mailbox）   │
  │                       └─ 合併回主分支                    │
  │  ────────────────────────────────────────────────────────
  │
  │  Phase 5: Code Review Team ─────────────────────────────
  │  │                                                      │
  │  ├─► logic-reviewer    ──┐                              │
  │  ├─► style-reviewer    ──┤ 並行                         │
  │  └─► security-reviewer ──┤                              │
  │                          ├─ 互相分享發現（Mailbox）       │
  │                          └─ Leader 彙整 Review Report   │
  │  ────────────────────────────────────────────────────────
  │
  └── Phase 6: Final Report
```

### Subagent vs Agent Teams 選擇

| 階段 | 使用 Subagent | 使用 Agent Teams |
|------|:---:|:---:|
| Phase 2 spec（單一 Teammate） | ✅ | — |
| Phase 3 design（2 人並行 + 交叉審查） | — | ✅ |
| Phase 4 code（1~2 人並行） | 僅後端時 ✅ | 前端+後端時 ✅ |
| Phase 5 review（3 人並行 + 交叉分享） | — | ✅ |

---

## 顯示模式（Split Pane vs In-Process）

Agent Teams 支援兩種顯示模式：

| 模式 | 說明 | 需求 |
|------|------|------|
| **in-process**（預設） | 所有 Teammate 在主終端內運行，用 Shift+Down 切換 | 任何終端 |
| **split panes** | 每個 Teammate 有獨立窗格，可同時看到所有人的輸出 | tmux 或 iTerm2 |

### `teammateMode` 設定

預設值為 `"auto"`：如果已在 tmux session 中就自動使用 Split Pane，否則用 in-process。

若要覆蓋，在 settings.json 中設定 `teammateMode`：

```json
{
  "teammateMode": "tmux"
}
```

也可以用 CLI flag 強制單次工作階段的模式：

```bash
claude --teammate-mode in-process
```

### Leader 的環境檢查

Leader 在 Phase 3（第一個並行階段）啟動前，檢查目前環境：

```bash
# 是否已在 tmux session 中（auto 模式會自動判斷）
echo $TMUX
```

| 條件 | 動作 |
|------|------|
| 已在 tmux session 中 | `auto` 模式自動使用 Split Pane，不需額外設定 |
| 未在 tmux + tmux 已安裝 | 提示：「建議在 tmux session 中執行以獲得 Split Pane 視覺化（`tmux new-session -s feature`）。要繼續使用 in-process 模式嗎？[Y/n]」 |
| tmux 未安裝 | 不提示，使用 in-process 模式（Shift+Down 切換 Teammate） |

---

## Phase 1: Setup（Leader 直接執行）

Leader 直接執行此階段，不啟動 Teammate。

### 1-1. 讀取規格書

使用 Read tool 讀取使用者指定的檔案。若檔案不存在則終止。

從規格書中解析：
- **功能名稱**：取第一個 `#` 標題，若無則取檔名（去掉副檔名）
- **需求描述**：取全文或「需求描述」區塊
- **技術備註**：取「技術備註」區塊（若有）
- **驗收條件**：取「驗收條件」區塊（若有）

### 1-2. 確認執行計畫

向使用者展示計畫，**含 Token 成本預估**：

```
📄 規格書：doc/訂閱推播統計.md
📝 功能名稱：訂閱推播統計報表
📋 需求摘要：{前 3 行}

即將依序執行（預估 Teammate 數量影響 Token 成本）：

  Phase 1: ✅ feature-start      — Leader 建立 Notion + Git branch
  Phase 2: ✅ spec-analyst        — 技術規格書（1 Teammate）
  Phase 3: ✅ Design Team         — DB + 架構（2 Teammates 並行 + 交叉審查）
  Phase 4: ✅ Code Gen Team       — 程式碼骨架（1~2 Teammates，spec 判斷後決定）
  Phase 5: ✅ Code Review Team    — 品質審查（3 Teammates 並行 + 交叉分享）

  預估 Teammate 總數：7~8 個（約為單一 Session 的 7~8 倍 Token 用量）

確認執行？[Y/n]
```

若使用者有指定 `--stop-at`、`--skip`、`--no-review`、`--backend-only`，相應調整計畫展示。

### 1-3. 執行 feature-start

使用 Skill tool 觸發 `/feature-start`：
- 功能名稱作為參數
- 自動建立 Git branch
- **記錄 Notion 頁面 URL**，供後續 Teammates 使用

### 1-4. tmux 環境檢查

在進入 Phase 3 前，執行 tmux 智慧判斷（見上方「tmux 智慧判斷」章節）。

---

## Phase 2: Spec Analysis（單一 Teammate — Subagent）

使用 **Agent tool** 啟動 subagent（不需 Team，因為只有一位）。

**model**: `opus`
**name**: `spec-analyst`

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
4. 若專案有前端（JSP/Vue/React 等），掃描前端目錄結構了解前端技術棧
5. 產出技術規格書，包含：
   - 功能範圍（In Scope / Out of Scope）
   - API 端點設計（表格：端點、Method、請求參數/Body、回傳格式、說明）
   - 業務邏輯規則（每個 API 的核心邏輯、前置驗證、處理流程、邊界條件）
   - 錯誤處理策略（表格：錯誤場景、處理方式、HTTP 狀態碼）
   - 分層決策（哪些邏輯放哪一層）
   - 效能需求（預期資料量、分頁、快取）
6. 使用 notion-update-page 將規格寫入「📐 技術規格」區塊
7. 使用 notion-update-page 更新開發階段 = 「規格設計」

API 路徑、回傳格式、錯誤處理必須遵循專案既有慣例。
輸出使用繁體中文。

## 重要：回傳格式
完成後，你必須回傳以下三部分：
1. **摘要**：產出了幾個 API、幾項業務規則（一行）
2. **前端判斷**：以下格式（Leader 用來決定是否啟動前端 Coder）
   ```
   FRONTEND_REQUIRED: true/false
   FRONTEND_TECH: JSP/Vue/React/Thymeleaf/無（偵測到的前端技術棧）
   FRONTEND_PAGES: 頁面清單（如：統計查詢頁、報表匯出頁）
   DB_REQUIRED: true/false
   ```
3. **完整技術規格內容**：你寫入 Notion 的完整 Markdown 內容（用 ```markdown 包裹）

Leader 需要你的完整產出來傳遞給下一階段的 Teammate。
```

**Leader 收到回傳後**：
- 解析摘要 → 顯示進度
- 解析 `FRONTEND_REQUIRED` 和 `DB_REQUIRED` → 決定後續階段配置
- 保存完整技術規格內容為 `{spec_output}`

```
✅ [Phase 2] spec-analyst 完成 — 3 個 API、5 項業務規則
   前端判斷：需要（JSP，統計查詢頁 + 報表匯出頁）
   DB 判斷：需要
```

---

## Phase 3: Design Team（並行 + 交叉審查）

### 條件判斷

- 若 `DB_REQUIRED = false` 或使用者指定 `--skip db` → 只啟動 arch-designer（退化為 Subagent 模式）
- 若兩者都需要 → 啟動 Design Team（Agent Teams 模式）

### 建立 Design Team

使用 **TeamCreate** 建立團隊，包含 2 位 Teammate。

使用 **Agent tool** 並行啟動兩位 Teammate（`run_in_background: true`）：

---

#### db-designer

**model**: `opus`
**name**: `db-designer`
**run_in_background**: `true`

**Prompt**：

```
你是一位資深資料庫設計師，負責此次功能開發的資料庫設計。
你是 Design Team 的一員，完成後需要與 arch-designer 進行交叉審查。

## 你的任務
根據技術規格設計資料表結構、索引和遷移 SQL，寫入 Notion 並將完整內容回傳。

## 規格書原始需求（參考）
{規格書完整內容}

## 前一階段產出：技術規格書（由 Leader 提供）
{spec_output}

## Notion 頁面 URL
{Notion 頁面 URL}

## 執行步驟
1. 讀取設定檔
2. 讀取當前專案的 CLAUDE.md，識別 DB 類型（MSSQL / MySQL / PostgreSQL）
3. 使用 Glob 掃描現有 Entity/POJO（2-3 個範本），了解命名慣例、公共欄位、註解風格
4. 讀取 ~/.claude/rules/database.md（若存在），取得資料庫規範
5. 根據技術規格書，設計資料庫結構：
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
1. **摘要**：新增幾個表、幾個索引、SQL 檔案路徑（一行）
2. **完整 DB 設計內容**：你寫入 Notion 的完整 Markdown 內容（用 ```markdown 包裹）

Leader 需要你的完整產出來傳遞給下一階段的 Teammate。
```

---

#### arch-designer

**model**: `opus`
**name**: `arch-designer`
**run_in_background**: `true`

**Prompt**：

```
你是一位資深後端架構設計師，負責此次功能開發的分層架構設計。
你是 Design Team 的一員，完成後需要與 db-designer 進行交叉審查。

## 你的任務
根據技術規格產出分層架構設計，寫入 Notion 並將完整內容回傳。

## 規格書原始需求（參考）
{規格書完整內容}

## 前一階段產出：技術規格書（由 Leader 提供）
{spec_output}

## Notion 頁面 URL
{Notion 頁面 URL}

## 執行步驟
1. 讀取設定檔
2. 讀取當前專案的 CLAUDE.md，識別架構模式和分層規則
3. 使用 Glob 掃描專案 package 結構，識別各層位置和慣例
4. 讀取 1-2 個現有功能的完整呼叫鏈（Controller → Service → DAO）
5. 讀取 ~/.claude/rules/design-patterns.md（若存在）
6. 根據技術規格，產出架構設計：
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
1. **摘要**：設計了幾個類別、使用了哪些設計模式（一行）
2. **完整架構設計內容**：你寫入 Notion 的完整 Markdown 內容（用 ```markdown 包裹）

Leader 需要你的完整產出來傳遞給下一階段的 Teammate。
```

---

### 交叉審查（Cross Review）

兩位 Teammate 都完成後，Leader 收集 `{db_output}` 和 `{arch_output}`，啟動交叉審查階段。

使用 **SendMessage** 讓兩位 Teammate 互相審查：

**發送給 db-designer**：
```
arch-designer 已完成架構設計，以下是他的產出。
請從 DB 設計的角度審查：
1. 架構中引用的表和欄位是否與你設計的一致？
2. Service 層的查詢需求是否有對應的索引？
3. 是否有遺漏的資料關聯？

{arch_output}

請回覆你的審查意見（簡要，3-5 點），以及是否需要調整你的 DB 設計。
若需要調整，請同時更新 Notion 和 SQL 檔案。
```

**發送給 arch-designer**：
```
db-designer 已完成 DB 設計，以下是他的產出。
請從架構設計的角度審查：
1. 表結構是否支撐所有 Service 方法的需求？
2. 是否有不必要的表或欄位？
3. 資料型別和命名是否與架構設計一致？

{db_output}

請回覆你的審查意見（簡要，3-5 點），以及是否需要調整你的架構設計。
若需要調整，請同時更新 Notion。
```

**Leader 等待兩邊回覆**，合併修正後的 `{db_output}` 和 `{arch_output}`。

```
✅ [Phase 3] Design Team 完成
   db-designer：2 個表、4 個索引 → doc/訂閱推播統計.sql
   arch-designer：8 個類別、策略模式
   交叉審查：db 調整 1 處索引、arch 無調整
```

---

## Phase 4: Code Gen Team（並行，條件性）

### 條件判斷

根據 Phase 2 spec-analyst 的 `FRONTEND_REQUIRED` 判斷：

| 條件 | 動作 |
|------|------|
| `FRONTEND_REQUIRED = false` 或 `--backend-only` | 只啟動 backend-coder（Subagent 模式） |
| `FRONTEND_REQUIRED = true` | 啟動 Code Gen Team（backend-coder + frontend-coder 並行） |

### 並行隔離策略

當有兩位 Coder 時，使用 **Git WorkTree** 避免檔案衝突：

```bash
# Leader 建立 WorkTree（或指示 Teammate 使用 isolation: worktree）
# backend-coder：在主分支工作
# frontend-coder：在獨立 WorkTree 工作
```

兩位 Coder 使用 Agent tool 的 `isolation: "worktree"` 參數，各自在隔離的工作目錄中產生程式碼。

### 建立 Code Gen Team

---

#### backend-coder

**model**: `opus`
**name**: `backend-coder`
**run_in_background**: `true`（若有 frontend-coder 時）

**Prompt**：

```
你是一位嚴謹的後端程式碼產生器。最重要的原則：產出的程式碼風格必須與專案現有程式碼完全一致。
{如果有 frontend-coder："你是 Code Gen Team 的後端成員，frontend-coder 會同時進行前端開發。"}

## 你的任務
根據前面階段的設計產出，按專案既有風格產生後端程式碼骨架。

## 前面階段產出：技術規格書（由 Leader 提供）
{spec_output}

## 前面階段產出：資料庫設計（由 Leader 提供）
{db_output}

## 前面階段產出：架構設計（由 Leader 提供）
{arch_output}

## Notion 頁面 URL
{Notion 頁面 URL}

## 執行步驟
1. 讀取設定檔，取得技術棧 ID
2. 根據技術棧 ID 掃描專案中同類型的現有程式碼各一個作為風格範本：
   - POJO/Entity、Mapper/Repository、Mapper XML（若適用）、Service、Controller
3. 讀取每個範本的完整內容，學習 package、import 順序、註解風格、縮排
4. 根據架構設計（類別清單、介面定義）和 DB 設計（CREATE TABLE），產生所有後端程式碼檔案
5. 要求：
   - package、import 順序、註解風格、縮排與範本完全一致
   - Getter/Setter 風格從範本學習（Lombok? 手動?）
   - Service 方法骨架含 TODO 註解標記待實作的業務邏輯
   - 註解和說明使用繁體中文
6. 先展示檔案清單表格（#、檔案路徑、類型、說明）{dry_run_instruction}
7. 使用 Write tool 依序建立檔案（先 Entity → Mapper → Service → Controller）
8. 使用 notion-update-page 將檔案清單寫入「📁 程式碼清單」的「後端」子區塊
9. 使用 notion-update-page 更新開發階段 = 「開發中」

## 重要：回傳格式
1. **摘要**：建立了幾個後端檔案（一行）
2. **API 端點清單**：列出所有 Controller 端點的 URL + Method（供 frontend-coder 參考）
3. **完整檔案清單**：表格（#、檔案路徑、類型、說明）
```

---

#### frontend-coder（條件性）

僅在 `FRONTEND_REQUIRED = true` 且未指定 `--backend-only` 時啟動。

**model**: `opus`
**name**: `frontend-coder`
**run_in_background**: `true`
**isolation**: `worktree`

**Prompt**：

```
你是一位嚴謹的前端程式碼產生器。最重要的原則：產出的前端程式碼風格必須與專案現有程式碼完全一致。
你是 Code Gen Team 的前端成員，backend-coder 正在同時進行後端開發。

## 你的任務
根據技術規格和架構設計，按專案既有前端風格產生前端頁面骨架。

## 前端技術棧
{FRONTEND_TECH}（由 spec-analyst 偵測）

## 需要的前端頁面
{FRONTEND_PAGES}

## 前面階段產出：技術規格書（由 Leader 提供）
{spec_output}

## 前面階段產出：架構設計（由 Leader 提供，含 API 端點設計）
{arch_output}

## Notion 頁面 URL
{Notion 頁面 URL}

## 執行步驟
1. 掃描專案前端目錄結構，辨識技術棧和檔案組織方式：
   - JSP 專案：`Glob **/*.jsp`、`Glob **/js/*.js`、`Glob **/css/*.css`
   - Vue 專案：`Glob **/src/views/**/*.vue`、`Glob **/src/components/**/*.vue`
   - React 專案：`Glob **/src/pages/**/*.tsx`、`Glob **/src/components/**/*.tsx`
2. 讀取 2-3 個現有前端頁面作為風格範本
3. 根據技術規格的 API 端點設計，產出前端頁面：
   - 頁面結構（HTML/JSP/Vue template）
   - API 呼叫邏輯（AJAX/fetch/axios）
   - 表單驗證（前端基本驗證）
   - 表格/圖表展示（若有資料列表需求）
4. 要求：
   - 頁面佈局、CSS class 命名、JS 風格與範本一致
   - API URL 和請求格式與技術規格一致
   - 包含 TODO 註解標記待細化的 UI 邏輯
   - 註解使用繁體中文
5. 先展示檔案清單 {dry_run_instruction}
6. 使用 Write tool 建立檔案

## 重要：回傳格式
1. **摘要**：建立了幾個前端檔案、使用了哪些前端技術（一行）
2. **API 依賴清單**：列出前端呼叫的所有 API 端點（供交叉確認）
3. **完整檔案清單**：表格（#、檔案路徑、類型、說明）
```

---

### API 契約確認（Code Gen Team 交叉審查）

若有 frontend-coder，兩位 Coder 都完成後，Leader 收集雙方的 API 清單進行比對：

**Leader 自行比對**（不需再啟動 Teammate）：
1. backend-coder 的「API 端點清單」
2. frontend-coder 的「API 依賴清單」

- **完全一致** → 通過
- **有差異** → 列出差異並使用 SendMessage 通知需要調整的一方修正

完成後合併 WorkTree：

```bash
# frontend-coder 的 WorkTree 變更合併回主分支
# Leader 或 Teammate 執行 merge
```

Leader 保存 `{code_output}`（兩方的檔案清單合併）。

```
✅ [Phase 4] Code Gen Team 完成
   backend-coder：6 個後端檔案（Entity / Mapper / Service / Controller）
   frontend-coder：3 個前端檔案（JSP / JS / CSS）
   API 契約確認：5 個端點全部一致 ✓
```

---

## Phase 5: Code Review Team（並行 + 交叉分享）

### 條件判斷

- 若使用者指定 `--no-review` 或 `--stop-at code` → 跳過此階段
- 否則 → 啟動 Code Review Team

### 建立 Review Team

使用 **TeamCreate** 建立團隊，包含 3 位 Reviewer。三位同時啟動（`run_in_background: true`）。

---

#### logic-reviewer

**model**: `opus`
**name**: `logic-reviewer`
**run_in_background**: `true`

**Prompt**：

```
你是一位嚴謹的邏輯正確性審查員，負責檢查程式碼的邏輯完整性。
你是 Code Review Team 的成員，完成後需與 style-reviewer 和 security-reviewer 分享發現。

## 你的任務
審查本次功能開發產生的所有程式碼，檢查邏輯正確性。

## 技術規格（參考）
{spec_output}

## DB 設計（參考）
{db_output}

## 架構設計（參考）
{arch_output}

## 程式碼檔案清單
{code_output}

## 執行步驟
1. 讀取專案 CLAUDE.md，了解架構慣例
2. 依序讀取 code_output 中列出的所有檔案
3. 檢查以下項目：
   - [ ] API 端點參數驗證是否完整
   - [ ] Service 業務邏輯是否符合技術規格中的規則
   - [ ] 查詢條件是否正確（SQL WHERE、分頁邏輯）
   - [ ] 例外處理是否覆蓋所有錯誤場景
   - [ ] 邊界條件處理（空值、空集合、超出範圍）
   - [ ] 回傳格式是否與技術規格一致
4. 對每個問題標記嚴重程度：🔴 嚴重 / 🟡 建議 / 🟢 良好

## 回傳格式
1. **摘要**：通過/需改進、發現幾個問題（一行）
2. **完整審查報告**（Markdown 表格：檔案、行號、問題、嚴重度、建議修正）
```

---

#### style-reviewer

**model**: `sonnet`（風格檢查不需 opus 等級）
**name**: `style-reviewer`
**run_in_background**: `true`

**Prompt**：

```
你是一位程式碼風格審查員，負責確保程式碼風格與專案一致。
你是 Code Review Team 的成員，完成後需與 logic-reviewer 和 security-reviewer 分享發現。

## 你的任務
審查本次功能開發產生的所有程式碼，檢查風格一致性。

## 程式碼檔案清單
{code_output}

## 執行步驟
1. 讀取專案 CLAUDE.md，了解命名規範和風格要求
2. 使用 Glob 掃描專案中 2-3 個同類型的現有檔案作為風格基準
3. 依序讀取新產生的檔案，逐一比對基準：
   - [ ] package 宣告、import 順序是否與基準一致
   - [ ] 類別/方法命名是否符合專案慣例
   - [ ] 註解風格（Javadoc? 行內註解?）是否一致
   - [ ] 縮排（tab/space）、括號風格是否一致
   - [ ] Lombok 使用方式是否一致
   - [ ] 常數定義方式是否一致
4. 對每個不一致標記：🟡 不一致 / 🟢 一致

## 回傳格式
1. **摘要**：風格一致性百分比（一行）
2. **完整審查報告**（Markdown 表格：檔案、問題、基準範例、建議修正）
```

---

#### security-reviewer

**model**: `opus`
**name**: `security-reviewer`
**run_in_background**: `true`

**Prompt**：

```
你是一位安全性與效能審查員，負責檢查程式碼的安全漏洞和效能問題。
你是 Code Review Team 的成員，完成後需與 logic-reviewer 和 style-reviewer 分享發現。

## 你的任務
審查本次功能開發產生的所有程式碼，檢查安全性和效能。

## 程式碼檔案清單
{code_output}

## 執行步驟
1. 讀取專案 CLAUDE.md，了解安全框架（ESAPI? AntiSamy?）
2. 依序讀取所有新產生的檔案，檢查：

   **安全性**：
   - [ ] SQL Injection（參數化查詢、MyBatis #{} vs ${}）
   - [ ] XSS（輸入過濾、輸出編碼）
   - [ ] 權限控制（是否有適當的 Session/權限檢查）
   - [ ] 敏感資料處理（密碼、Token 是否加密）
   - [ ] CSRF 防護

   **效能**：
   - [ ] N+1 查詢問題
   - [ ] 缺少分頁的大量查詢
   - [ ] 不必要的全表掃描（缺少索引）
   - [ ] 可快取但未快取的重複查詢
   - [ ] 迴圈內的 DB 呼叫

3. 對每個問題標記：🔴 安全漏洞 / 🟡 效能風險 / 🟢 良好

## 回傳格式
1. **摘要**：安全問題 N 個、效能問題 N 個（一行）
2. **完整審查報告**（Markdown 表格：檔案、行號、類型、問題、嚴重度、建議修正）
```

---

### 交叉分享（Review Team Mailbox）

三位 Reviewer 都完成後，Leader 使用 **SendMessage** 讓他們互相分享發現：

**發送給每位 Reviewer**：
```
其他兩位 Reviewer 的發現如下，請檢查是否有：
1. 你遺漏但他們發現的問題
2. 你們觀點衝突的地方
3. 需要從你的角度補充的交叉觀點

## logic-reviewer 的發現
{logic_review_summary}

## style-reviewer 的發現
{style_review_summary}

## security-reviewer 的發現
{security_review_summary}

請回覆：
1. 遺漏補充（若有）
2. 觀點衝突說明（若有）
3. 交叉觀點（如：邏輯問題可能導致安全風險等）
```

### Leader 彙整 Review Report

收集三方審查 + 交叉補充，使用 **notion-update-page** 寫入 Notion「📋 程式碼審查」區塊：

```markdown
## 📋 程式碼審查報告

### 審查摘要
| 審查項 | 結果 | 問題數 |
|--------|------|--------|
| 邏輯正確性 | ✅ 通過 / ⚠️ 需改進 | N 個 |
| 程式碼風格 | ✅ 95% 一致 | N 個 |
| 安全性與效能 | ✅ 通過 / ⚠️ 需改進 | N 個 |

### 🔴 必須修正（嚴重問題）
...

### 🟡 建議改進
...

### 交叉審查補充
...
```

更新開發階段 = 「程式碼審查」。

```
✅ [Phase 5] Code Review Team 完成
   logic-reviewer：通過，1 個建議
   style-reviewer：92% 一致，3 處不一致
   security-reviewer：0 個安全漏洞，1 個效能建議
   交叉補充：2 個新發現
```

---

## Phase 6: Final Report（Leader）

所有階段完成後，Leader 向使用者展示完整報告：

```
🎉 功能開發全流程完成！

📄 規格書：doc/訂閱推播統計.md
📝 功能名稱：訂閱推播統計報表
🔀 Git branch：feature/subscription-push-statistics
📊 Notion 頁面：https://notion.so/xxx

已完成階段：
  ✅ Phase 1  feature-start     — Notion 條目 + Git branch
  ✅ Phase 2  spec-analyst      — 3 個 API、5 項業務規則
  ✅ Phase 3  Design Team       — 2 個表 + 8 個類別（交叉審查通過）
  ✅ Phase 4  Code Gen Team     — 6 個後端 + 3 個前端（API 契約一致）
  ✅ Phase 5  Code Review Team  — 通過（1 個效能建議）

Teammate 統計：
  總計 8 個 Teammates（spec:1 + design:2 + code:2 + review:3）

後續操作：
  1. 修正 Review 報告中的 🟡 建議項目
  2. 實作 Service 中的 TODO 業務邏輯
  3. 隨時用 /feature-update 記錄進度
  4. 最後用 /feature-close 結案
```

---

## 錯誤處理

### 單一 Teammate 失敗

若某個 Teammate 執行失敗，Leader 不自動終止整個流程：

```
❌ db-designer 失敗：Notion API 回應 429 (Rate Limit)

選擇：
1. 重試此 Teammate
2. 跳過，繼續下一階段（下游 Agent 的 prompt 中標記已跳過）
3. 終止流程（已完成的階段不受影響）
```

### 整個 Team 失敗

若 Team 中所有 Teammate 都失敗：
- 顯示每個 Teammate 的錯誤摘要
- 建議手動執行對應的個別 Skill（`/feature-db`、`/feature-arch` 等）

### Agent Teams 未啟用

若環境變數未設定，顯示完整的設定指引：

```
⚠️ Agent Teams 功能未啟用。請完成以下設定：

1. 設定環境變數（擇一）：
   a. 加入 ~/.zshrc：export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
   b. 加入 ~/.claude/settings.json 的 env 區塊

2. 重啟 Claude Code

3.（建議）安裝 tmux 以獲得 Split Pane 視覺化：
   brew install tmux
```

### 交叉審查發現嚴重問題

若 Design Team 交叉審查發現 🔴 嚴重不一致：

```
⚠️ Design Team 交叉審查發現嚴重問題：
  db-designer：表 X 缺少 arch-designer 需要的欄位 Y

選擇：
1. 讓 db-designer 修正後重新交叉審查
2. 忽略，繼續下一階段
3. 終止流程
```

### 中途終止後恢復

已完成的階段不受影響（已寫入 Notion）。使用者可手動執行剩餘的 skill：

```
已完成 start + spec + design，若要繼續：
  /feature-scaffold     ← 產生程式碼（會從 Notion 讀取設計產出）
  /feature-review       ← 或直接做 Code Review
```

---

## 邊界情況

- **規格書檔案不存在**：顯示錯誤訊息，提示檢查路徑
- **規格書為空或內容過短（< 10 字）**：警告並詢問是否繼續
- **規格書過長（> 500 行）**：正常處理，Agent 會自行擷取重點
- **設定檔不存在**：提示使用者先執行 `/feature-setup`
- **非 Git repo**：跳過 branch 建立和 WorkTree，其餘正常
- **使用者在確認時取消**：直接終止
- **Agent Teams 未啟用**：顯示設定指引
- **前端技術棧無法偵測**：詢問使用者確認或指定 `--backend-only`
- **WorkTree 建立失敗**：退化為串行模式（backend-coder 先，frontend-coder 後）
- **Teammate 回傳過大**：優先壓縮規格書原文（spec_output 已涵蓋重點）
- **Token 用量接近上限**：提示使用者，建議用 `--no-review` 或 `--backend-only` 減少 Teammate 數量
