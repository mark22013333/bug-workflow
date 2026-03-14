---
name: feature-auto
description: 讀取本地規格書檔案，以 Agent Teams 模式自動完成功能開發全流程（規格 → 設計 → 程式碼 → 審查）。支援並行開發、交叉審查、tmux 分割視窗。當使用者提到「feature-auto」、「自動開發」、「讀取規格書」、「匯入規格」時觸發此 Skill。
---

# feature-auto — Agent Teams 驅動的全流程自動化功能開發

讀取本地規格書檔案，以 **Agent Teams** 模式驅動全流程自動化。並行階段使用真正的 Agent Teams（Teammates 有獨立 Context Window，透過 Mailbox 直接通訊），在 tmux 環境下自動啟用 Split Pane。

---

## 前置條件

### 環境變數

必須啟用 Agent Teams 實驗功能（擇一設定）：

**方式 A**：加入 shell profile（`~/.zshrc` 或 `~/.bashrc`）
```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

**方式 B**：加入 settings.json 的 `env` 區塊
```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

> **settings.json 路徑**：依使用者的啟動方式而定。若設定了 `CLAUDE_CONFIG_DIR` 環境變數，settings.json 位於 `$CLAUDE_CONFIG_DIR/settings.json`；否則位於 `~/.claude/settings.json`。

### 設定檔

feature-workflow 設定檔，依序檢查：
1. `~/.claude-company/feature-workflow-config.md`
2. `~/.claude/feature-workflow-config.md`

若都不存在，提示使用者先執行 `/feature-setup`。

### 顯示模式（Split Pane）

Agent Teams 預設 `teammateMode` 為 `"auto"`：
- **已在 tmux session 中** → 自動使用 Split Pane（每個 Teammate 獨立窗格）
- **未在 tmux 中** → 使用 in-process 模式（Shift+Down 切換 Teammate）

要獲得 Split Pane 效果，只需在 tmux 中啟動 Claude Code：
```bash
tmux new-session -s feature
claude  # 或 claudec 等自訂指令
```

---

## 使用方式

```
/feature-auto <規格書檔案路徑> [選項]
```

### 參數

| 參數 | 說明 |
|------|------|
| `<規格書檔案路徑>` | 必填，本地 Markdown / 純文字檔案路徑 |
| `--stop-at <階段>` | 執行到指定階段後暫停 |
| `--dry-run` | code 階段僅預覽，不建立檔案 |
| `--skip <階段,階段>` | 跳過指定階段（逗號分隔） |
| `--backend-only` | 強制跳過前端 Coder |
| `--no-review` | 跳過 Code Review 階段 |

### 階段列表

| 階段 ID | 說明 | 模式 |
|---------|------|------|
| `start` | 建立 Notion 條目 + Git branch | Leader 直接執行 |
| `spec` | 技術規格書 | 單一 Subagent |
| `design` | DB 設計 + 架構設計 | **Agent Teams（2 人並行 + 交叉審查）** |
| `code` | 程式碼骨架產生 | **Agent Teams（後端 + 前端並行）** |
| `review` | 程式碼品質審查 | **Agent Teams（3 人並行 + 交叉分享）** |

### 範例

```bash
/feature-auto doc/訂閱推播統計.md                      # 完整流程
/feature-auto doc/訂閱推播統計.md --stop-at design      # 只到設計階段
/feature-auto doc/訂閱推播統計.md --skip db             # 跳過 DB 設計
/feature-auto doc/訂閱推播統計.md --backend-only        # 只產後端
/feature-auto doc/訂閱推播統計.md --no-review --dry-run # 跳過審查 + 預覽
```

---

## 規格書格式

無強制格式，建議包含：

```markdown
# 功能名稱

## 需求描述
- 功能目的、目標使用者、使用情境

## 驗收條件
- [ ] 條件 1

## 技術備註（選填）
- API 路徑偏好、是否需要前端頁面、DB 設計偏好
```

---

## 整體架構

```
Leader（delegate mode — 只協調不寫 code）
  │
  │  Phase 1: feature-start（Leader 直接執行）
  │
  ├── Phase 2: spec-analyst（Subagent，產出寫入 doc/{功能名稱}-spec.md）
  │     判斷：是否需要前端？是否需要 DB？
  │
  │  Phase 3: Design Team ── Agent Teams ──────────────
  │  ├── db-designer  ──┐ 並行，各自寫入 Notion
  │  └── arch-designer ─┤ 完成後互相 Review（Mailbox）
  │  ─────────────────────────────────────────────────
  │
  │  Phase 4: Code Gen Team ── Agent Teams ────────────
  │  ├── backend-coder  ──┐ 並行
  │  └── frontend-coder ──┤ 完成後確認 API 契約（Mailbox）
  │       (條件性)         │ 各用不同檔案目錄避免衝突
  │  ─────────────────────────────────────────────────
  │
  │  Phase 5: Code Review Team ── Agent Teams ─────────
  │  ├── logic-reviewer    ──┐
  │  ├── style-reviewer    ──┤ 並行，完成後互相分享發現
  │  └── security-reviewer ──┘ Leader 彙整 Review Report
  │  ─────────────────────────────────────────────────
  │
  └── Phase 6: Final Report
```

### 上下文傳遞方式

各階段的產出寫入 **Notion** 和本地 **doc/** 檔案，下游 Teammates 從 Notion 和檔案讀取：

| 產出 | Notion 區塊 | 本地檔案 |
|------|------------|---------|
| 技術規格 | 📐 技術規格 | `doc/{功能名稱}-spec.md` |
| DB 設計 | 🗄️ 資料庫設計 | `doc/{功能名稱}.sql` |
| 架構設計 | 🏗️ 架構設計 | （僅 Notion） |
| 程式碼清單 | 📁 程式碼清單 | 實際程式碼檔案 |
| 審查報告 | 📋 程式碼審查 | （僅 Notion） |

---

## Phase 1: Setup（Leader 直接執行）

Leader 不啟動 Teammate，直接完成：

### 1-1. 讀取規格書

使用 Read tool 讀取檔案。從中解析功能名稱、需求描述、技術備註、驗收條件。

### 1-2. 確認執行計畫

向使用者展示計畫，含 Token 成本預估和各階段 Teammate 數量。若有 `--stop-at`、`--skip` 等選項，相應調整展示。

### 1-3. 執行 feature-start

使用 Skill tool 觸發 `/feature-start`，記錄 Notion 頁面 URL。

### 1-4. 環境檢查

檢查是否在 tmux session 中：
- **是** → 告知使用者「偵測到 tmux，並行階段將使用 Split Pane」
- **否 + tmux 已安裝** → 提示「建議在 tmux session 中執行以獲得 Split Pane（`tmux new-session -s feature`）。要繼續使用 in-process 模式嗎？」
- **tmux 未安裝** → 不提示，使用 in-process 模式

---

## Phase 2: Spec Analysis（Subagent）

Phase 2 只有一位，使用 **Agent tool**（Subagent）即可。

使用 Agent tool 啟動 subagent，model 為 opus，指示其：

1. 讀取 feature-workflow 設定檔（依序檢查 `~/.claude-company/` 和 `~/.claude/`）
2. 讀取專案 CLAUDE.md，了解技術棧和架構慣例
3. 掃描專案中相關的 Controller 和 Service（1-2 個），了解 API 風格
4. 若專案有前端（JSP/Vue/React），掃描前端目錄結構
5. 產出完整技術規格書（API 設計、業務邏輯、錯誤處理、分層決策、效能需求）
6. 寫入 Notion「📐 技術規格」區塊
7. 使用 Write tool 將完整規格寫入 `doc/{功能名稱}-spec.md`
8. **回傳時必須包含前端/DB 判斷**：
   - `FRONTEND_REQUIRED: true/false`
   - `FRONTEND_TECH: JSP/Vue/React/無`
   - `FRONTEND_PAGES: 頁面清單`
   - `DB_REQUIRED: true/false`

**Leader 收到回傳後**：
- 解析前端/DB 判斷 → 決定後續階段配置
- 顯示進度

---

## Phase 3: Design Team（Agent Teams — 並行 + 交叉審查）

### 條件判斷

- `DB_REQUIRED = false` 或 `--skip db` → 只需 arch-designer（退化為 Subagent）
- 兩者都需要 → **建立 Agent Teams**

### 建立團隊

**使用自然語言要求 Claude 建立 Agent Team**（不要使用 Agent tool）：

```
建立一個 Agent Team 來做設計工作，生成 2 個 Teammate：

1. db-designer — 資料庫設計師
   - 讀取 doc/{功能名稱}-spec.md 取得技術規格
   - 讀取專案 CLAUDE.md 識別 DB 類型（MSSQL / MySQL）
   - 掃描現有 Entity/POJO 2-3 個了解命名慣例
   - 設計 CREATE TABLE、CREATE INDEX、範例資料、Rollback SQL
   - 寫入 Notion「🗄️ 資料庫設計」區塊
   - 輸出 SQL 到 doc/{功能名稱}.sql
   - 使用繁體中文

2. arch-designer — 架構設計師
   - 讀取 doc/{功能名稱}-spec.md 取得技術規格
   - 讀取專案 CLAUDE.md 識別架構模式和分層規則
   - 掃描 1-2 個現有功能的呼叫鏈（Controller → Service → DAO）
   - 產出分層架構圖（Mermaid）、類別清單、介面定義、設計模式選擇
   - 寫入 Notion「🏗️ 架構設計」區塊
   - 使用繁體中文

任務依賴：兩位同時開始，無依賴。
完成後：請他們互相審查對方的設計（db-designer 審查架構、arch-designer 審查 DB），
透過 Mailbox 分享發現，確認表結構與架構設計一致後回報。

Notion 頁面 URL：{Notion 頁面 URL}
使用 delegate mode，我只協調不寫 code。
```

### 交叉審查

Agent Teams 的 Teammates 可透過 Mailbox 直接通訊。Leader 在兩位完成主要工作後，要求他們互相 Review：

- db-designer 審查 arch-designer 的設計：表和欄位是否對應、查詢需求是否有索引
- arch-designer 審查 db-designer 的設計：表結構是否支撐 Service 方法、命名是否一致

交叉審查完成後，Leader 確認結果，清理 Design Team。

---

## Phase 4: Code Gen Team（Agent Teams — 並行，條件性）

### 條件判斷

| 條件 | 動作 |
|------|------|
| `FRONTEND_REQUIRED = false` 或 `--backend-only` | 只需 backend-coder（Subagent 即可） |
| `FRONTEND_REQUIRED = true` | **建立 Agent Teams**（backend-coder + frontend-coder 並行） |

### 僅後端（Subagent）

使用 Agent tool 啟動 subagent，指示其：
- 讀取設定檔取得技術棧 ID
- 從 Notion 讀取 DB 設計和架構設計
- 掃描專案中同類型的現有程式碼各一個作為風格範本
- 按架構設計的類別清單產生所有後端程式碼骨架
- 風格必須與專案完全一致
- 寫入 Notion「📁 程式碼清單」區塊

### 前端 + 後端（Agent Teams）

**使用自然語言要求 Claude 建立 Agent Team**：

```
建立一個 Agent Team 來產生程式碼，生成 2 個 Teammate：

1. backend-coder — 後端程式碼產生器
   - 從 Notion 頁面讀取 DB 設計和架構設計
   - 讀取設定檔取得技術棧 ID
   - 掃描專案現有程式碼學習風格（POJO、Mapper、Service、Controller 各一個範本）
   - 按架構設計產生所有後端程式碼骨架
   - 風格必須與專案完全一致（package、import 順序、註解、縮排）
   - Service 方法含 TODO 標記待實作邏輯
   - 寫入 Notion「📁 程式碼清單」的「後端」區塊
   - 完成後向 frontend-coder 分享 API 端點清單（URL + Method + 請求/回應格式）

2. frontend-coder — 前端程式碼產生器
   - 前端技術棧：{FRONTEND_TECH}
   - 需要的頁面：{FRONTEND_PAGES}
   - 從 Notion 頁面讀取技術規格中的 API 設計
   - 掃描專案前端目錄，讀取 2-3 個現有頁面作為風格範本
   - 產生前端頁面（HTML/JSP/Vue）、API 呼叫邏輯、表單驗證、表格展示
   - 風格必須與專案完全一致
   - 寫入 Notion「📁 程式碼清單」的「前端」區塊
   - 完成後向 backend-coder 分享 API 依賴清單

重要：兩位 Teammate 負責不同目錄的檔案（後端改 src/main/java，前端改 webapp/JSP/JS），
不會衝突。

任務依賴：兩位同時開始。
完成後：請他們互相確認 API 契約是否一致（端點 URL、參數、回應格式），
不一致的地方由 backend-coder 為準，frontend-coder 調整。
{dry_run_instruction}

Notion 頁面 URL：{Notion 頁面 URL}
使用 delegate mode。
```

> `{dry_run_instruction}`：若使用者指定 `--dry-run`，加入「只展示檔案清單，不實際建立檔案」。

API 契約確認完成後，Leader 清理 Code Gen Team。

---

## Phase 5: Code Review Team（Agent Teams — 並行 + 交叉分享）

### 條件判斷

`--no-review` 或 `--stop-at code` → 跳過。

### 建立團隊

**使用自然語言要求 Claude 建立 Agent Team**：

```
建立一個 Agent Team 來做 Code Review，生成 3 個 Reviewer：

1. logic-reviewer — 邏輯正確性審查
   - 讀取專案 CLAUDE.md 了解架構慣例
   - 從 Notion 頁面讀取技術規格和架構設計
   - 讀取本次新增的所有程式碼檔案
   - 檢查：API 參數驗證、業務邏輯正確性、查詢條件、例外處理、邊界條件、回傳格式
   - 標記嚴重程度：🔴 嚴重 / 🟡 建議 / 🟢 良好
   - 使用 Opus 模型

2. style-reviewer — 程式碼風格審查
   - 掃描專案中 2-3 個同類型的現有檔案作為風格基準
   - 比對新產生的檔案：package、import、命名、註解、縮排、Lombok
   - 標記：🟡 不一致 / 🟢 一致
   - 可使用 Sonnet 模型（風格檢查不需 Opus）

3. security-reviewer — 安全性與效能審查
   - 讀取專案 CLAUDE.md 了解安全框架（ESAPI? AntiSamy?）
   - 安全性：SQL Injection、XSS、權限控制、敏感資料、CSRF
   - 效能：N+1 查詢、缺少分頁、缺少索引、迴圈內 DB 呼叫
   - 標記：🔴 安全漏洞 / 🟡 效能風險 / 🟢 良好
   - 使用 Opus 模型

任務依賴：三位同時開始。
完成後：請他們互相分享各自的發現，檢查是否有遺漏、觀點衝突或交叉觀點
（如邏輯問題可能導致安全風險）。

最後由我彙整所有發現，產出完整的 Review Report 寫入 Notion「📋 程式碼審查」區塊。

Notion 頁面 URL：{Notion 頁面 URL}
使用 delegate mode。
所有輸出使用繁體中文。
```

Leader 等待交叉分享完成後，彙整 Review Report 寫入 Notion，清理 Review Team。

---

## Phase 6: Final Report（Leader）

所有階段完成後，Leader 展示完整報告：

```
🎉 功能開發全流程完成！

📄 規格書：{路徑}
📝 功能名稱：{名稱}
🔀 Git branch：{branch}
📊 Notion 頁面：{URL}

已完成階段：
  ✅ Phase 1  feature-start     — Notion 條目 + Git branch
  ✅ Phase 2  spec-analyst      — N 個 API、N 項業務規則
  ✅ Phase 3  Design Team       — N 個表 + N 個類別（交叉審查通過）
  ✅ Phase 4  Code Gen Team     — N 個後端 + N 個前端（API 契約一致）
  ✅ Phase 5  Code Review Team  — 通過（N 個建議）

後續操作：
  1. 修正 Review 報告中的 🟡 建議項目
  2. 實作 Service 中的 TODO 業務邏輯
  3. 隨時用 /feature-update 記錄進度
  4. 最後用 /feature-close 結案
```

---

## 錯誤處理

### Teammate 失敗

Leader 不自動終止流程，提供選項：重試 / 跳過 / 終止。

### Agent Teams 未啟用

顯示設定指引，說明兩種設定方式（shell profile 或 settings.json），並提示 settings.json 路徑取決於 `CLAUDE_CONFIG_DIR` 環境變數。

### 交叉審查發現嚴重問題

提供選項：修正後重新審查 / 忽略繼續 / 終止。

### 中途終止後恢復

已完成的階段不受影響（已寫入 Notion）。提示使用者可手動執行個別 Skill 繼續。

---

## 邊界情況

- **規格書不存在**：終止並提示
- **規格書過短（< 10 字）**：警告並詢問
- **設定檔不存在**：提示 `/feature-setup`
- **非 Git repo**：跳過 branch 建立，其餘正常
- **前端技術棧無法偵測**：詢問使用者或 `--backend-only`
- **Token 用量接近上限**：建議 `--no-review` 或 `--backend-only`
- **每個工作階段只能一個 Team**：Phase 3 的 Team 必須清理後才能建立 Phase 4 的 Team
- **Teammates 不能嵌套 Team**：只有 Leader 可以建立 Team
- **Session 不能 Resume**：Agent Teams 的已知限制，中斷後需重新建立
