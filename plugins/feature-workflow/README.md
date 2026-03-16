# Feature Workflow Plugin

功能開發工作流 — 整合 Notion 與 Claude Code，涵蓋需求分析、規格設計、DB 設計、架構設計、程式碼產生、驗收驗證、品質審查到結案的完整生命週期管理。

此 Plugin 不綁定特定專案架構，所有 Agent/Skill 執行時會讀取當前專案的 CLAUDE.md 動態適配技術棧、分層慣例和程式碼風格。

## 安裝

```bash
claude plugin marketplace add mark22013333/crew && \
claude plugin install feature-workflow && \
claude plugin enable feature-workflow
```

首次使用前執行 `/feature-setup` 完成設定引導。

### 更新

```bash
claude plugin update feature-workflow@company-marketplace
```

更新完成後**重啟 Claude Code** 使新版生效。

> 若 `update` 顯示已是最新但功能未生效，可先移除再重裝：
> ```bash
> claude plugin uninstall feature-workflow@company-marketplace && \
> claude plugin install feature-workflow@company-marketplace
> ```

---

## 兩種模式

### Notion 驅動模式（feature-*）

每步直接讀寫 Notion，適合需要即時協作的場景。

```
/feature-setup (首次)
       │
/feature-start ── /feature-spec ── /feature-db ── /feature-arch
       │                                              │
       │            /feature-update (隨時穿插)          │
       │                                              ▼
       └──────────────── /feature-scaffold ── /feature-review ── /feature-close
```

### 本地規劃模式（plan-*，v3）

設計文件存在 `.spec/{slug}/` 目錄，僅建立和結案時呼叫 Notion，大幅減少 API 呼叫。

```
/plan-start ── /plan ── /plan-build ── /plan-verify ── /plan-review ── /plan-close
                                           │
                                    [IDE 啟動本地服務]
                                    [Chrome 開啟頁面]
```

---

## Skill 清單

### feature-* Notion 驅動

| Skill | 說明 |
|-------|------|
| `/feature-setup` | 首次設定引導（含 Agent 安裝選項） |
| `/feature-start` | 建立 Notion 頁面 + Git branch |
| `/feature-spec` | 技術規格書（Agent, opus） |
| `/feature-db` | 資料庫設計（Agent, opus） |
| `/feature-arch` | 架構設計（Agent, opus） |
| `/feature-scaffold` | 程式碼骨架產生（Agent, opus） |
| `/feature-update` | 更新開發進度 |
| `/feature-review` | 程式碼品質檢查 |
| `/feature-close` | 結案 + 同步設計庫 |
| `/feature-auto` | 讀取規格書自動執行完整流程（Agent Teams） |
| `/feature-stack` | 偵測專案分層結構，建立自訂技術棧 |
| `/project-add` | 新增或更新專案對應（來自 bug-workflow） |

### plan-* 本地規劃

| Skill | 說明 | Notion 呼叫 |
|-------|------|-------------|
| `/plan-start` | 建立任務到 .spec/ + Notion | **2-3 次** |
| `/plan` | 本地規劃（spec/db/arch） | **0 次** |
| `/plan-build` | Agent Teams 產生程式碼 | **0 次** |
| `/plan-verify` | chrome-cdp 操作瀏覽器驗證驗收條件 | **0 次** |
| `/plan-review` | Agent Teams 3 人程式碼審查 | **0 次** |
| `/plan-close` | 批次同步 Notion + Git 提交 | **3-5 次** |
| `/plan-sync` | 手動同步 .spec/ 到 Notion | **2-3 次** |
| `/plan-status` | 查看任務狀態 | **0 次** |

---

## 前置設定

### Agent Teams（plan-build / plan-review / feature-auto）

```json
// ~/.claude/settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

> tmux session 中自動啟用 Split Pane，只需 `tmux new-session -s dev` 後啟動 Claude Code。

### Chrome Remote Debugging（plan-verify）

`/plan-verify` 透過 Chrome DevTools Protocol 連接使用者已開啟的 Chrome，直接操作已登入的 session 驗證驗收條件。對需要 SSO/VPN 的內部系統特別有用。

**前置條件**：

| 項目 | 需求 |
|------|------|
| Node.js | **22 以上**（cdp.mjs 使用內建 WebSocket） |
| Chrome | 啟用 Remote Debugging |

**啟用 Chrome Remote Debugging**：

1. 在 Chrome 網址列輸入 `chrome://inspect/#remote-debugging`
2. 開啟「Remote debugging」切換開關
3. 確認完成：執行 `/plan-verify`，若前置檢查通過即可

> 也支援 Chromium、Brave、Edge、Vivaldi。

**使用方式**：

```bash
/plan-verify                    # 完整驗證所有驗收條件
/plan-verify --manual           # 互動模式，每步驟等待確認
/plan-verify <URL>              # 指定目標頁面
/plan-verify --api-only         # 只驗證 API（不需 Chrome）
/plan-verify --recheck          # 僅重新驗證上次失敗的項目
```

---

## 一鍵自動化（/feature-auto）

讀取規格書檔案，以 Agent Teams 自動完成全流程：

```bash
/feature-auto doc/訂閱推播統計.md                   # 完整流程
/feature-auto doc/spec.md --stop-at design          # 只到設計階段
/feature-auto doc/spec.md --backend-only            # 只產後端
/feature-auto doc/spec.md --no-review               # 跳過審查
/feature-auto doc/spec.md --dry-run                 # 預覽不建立檔案
```

### 規格書範本

規格書無強制格式，但結構化內容能讓 Agent 產出更精確：

```markdown
# 功能名稱

## 需求描述
- **功能目的**：...
- **目標使用者**：...
- **使用情境**：1. ... 2. ... 3. ...

## 驗收條件
- [ ] 條件一
- [ ] 條件二

## 技術備註（選填）
- API 路徑風格、資料來源、分頁限制等
```

> 規格書可以很簡短（三行需求描述），Agent 會根據專案 CLAUDE.md 自動推斷。

---

## 技術棧支援

### 內建

| ID | 框架 | ORM | scaffold 行為 |
|----|------|-----|--------------|
| `spring-mvc-mybatis` | Spring MVC 4.x | MyBatis + tk.mybatis | POJO + Mapper XML + Service(Interface+Impl) + Controller |
| `spring-boot-mybatis` | Spring Boot 2.x+ | MyBatis + tk.mybatis | Entity + Mapper + Service + Controller + DTO |
| `spring-boot-jpa` | Spring Boot 2.x+ | JPA/Hibernate | Entity + Repository + Service + Controller + DTO |
| `spring-boot-mybatis-plus` | Spring Boot 2.x+ | MyBatis-Plus | Entity + BaseMapper + Service(IService+Impl) + Controller |

### 自訂（/feature-stack）

適用於內建未涵蓋的框架。執行 `/feature-stack` 自動掃描專案結構產生掃描規則，也可指定 ID：`/feature-stack my-custom-stack`。

---

## Agent 雙模式

| 模式 | 說明 |
|------|------|
| **A. SKILL.md 內嵌**（預設） | 安裝 Plugin 即可用，Skill 執行時內嵌 prompt 啟動 opus Agent |
| **B. 獨立 Agent 檔案** | `/feature-setup` 時可選安裝到 `~/.claude-company/agents/`，可獨立使用 |

獨立 Agent：`spec-analyst`、`db-designer`、`backend-designer`、`code-generator`。

---

## 與 bug-workflow 的關係

- 共用 Notion「任務追蹤工具」和「專案資料庫」
- 共用 `/project-add` 管理專案對應
- setup 自動匯入 bug-workflow 的共用 ID
- 互不干擾，可同時使用

## 授權

MIT License
