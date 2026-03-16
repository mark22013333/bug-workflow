---
name: plan-build
description: 從 .spec/ 讀取設計文件，以 Agent Teams leader-delegate 模式產生程式碼。Leader 只協調不寫 code。當使用者提到「plan-build」、「build」、「產生程式碼」時觸發此 Skill。
---

# plan-build — Agent Teams 程式碼產生

從 `.spec/{slug}/` 讀取設計文件（spec.md、db.md、arch.md），以 **Agent Teams** leader-delegate 模式產生程式碼。Leader 只負責協調，不直接寫程式碼。

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

### 設計文件

建議至少完成 `/plan arch`（arch.md 存在），否則提示先執行 `/plan`。

---

## 使用方式

```
/plan-build                # 完整產生（後端 + 前端）
/plan-build --dry-run      # 預覽不建立檔案
/plan-build --backend-only # 只產後端
```

---

## 流程

### 1. 定位活躍任務

與 `/plan` 相同邏輯：從 Git branch 或 `_index.md` 匹配活躍任務。

讀取 `.spec/{slug}/README.md` 取得元資訊。

### 2. 讀取設計文件

讀取以下 `.spec/{slug}/` 下的檔案：

| 檔案 | 用途 | 必要性 |
|------|------|--------|
| `spec.md` | 技術規格（API 設計、業務邏輯） | 建議 |
| `db.md` | DB 設計（表結構） | 建議 |
| `db.sql` | SQL 檔案 | 選讀 |
| `arch.md` | 架構設計（類別清單、介面定義） | **強烈建議** |

若 arch.md 不存在，警告使用者並詢問是否繼續。

### 3. 判斷前端需求

從 `spec.md` 的「判斷」區塊讀取：
- `FRONTEND_REQUIRED`: true/false
- `FRONTEND_TECH`: JSP/Vue/React/無

若為 true 且未指定 `--backend-only`，將建立包含前端 Coder 的 Team。

### 4. 確認執行計畫

向使用者展示：

```
即將啟動 Agent Teams 產生程式碼：

📄 設計來源：.spec/{slug}/
📊 Teammate 配置：
  • backend-coder — 讀取 arch.md + db.md 產生後端程式碼
  {• frontend-coder — 讀取 spec.md 產生前端頁面}（條件性）

{--dry-run: 預覽模式，不建立檔案}

確認開始？[Y/n]
```

### 5. 讀取專案上下文（給 Teammates 的共用上下文）

收集一次，嵌入到 Team 建立指令中：

- 專案 CLAUDE.md 內容
- 技術棧 ID 和定義（從設定檔）
- 1-2 個現有程式碼範本（POJO、Mapper、Service、Controller 各一個）的檔案路徑

### 6. 啟動 Agent Teams

#### 僅後端（Subagent）

若 `FRONTEND_REQUIRED = false` 或 `--backend-only`，使用 **Agent tool** 啟動 subagent：

```
你是後端程式碼產生器。

## 設計文件
{arch.md 內容}

## DB 設計
{db.md 內容}

## 專案上下文
{CLAUDE.md 內容}

## 技術棧
{技術棧 ID 和定義}

## 現有程式碼範本
請讀取以下檔案作為風格參考：
{範本檔案路徑清單}

## 任務
按架構設計的類別清單，依序產生所有後端程式碼骨架：
1. POJO/Entity（含 Lombok、表註解）
2. Mapper/DAO（tk.mybatis 或 JPA Repository）
3. Mapper XML（若使用 MyBatis）
4. Service Interface
5. Service Impl（方法含 TODO 標記待實作邏輯）
6. Controller（含 @RequestMapping、參數驗證）

風格必須與專案完全一致（package、import 順序、註解、縮排）。
{dry_run_instruction}

使用繁體中文撰寫註解。
```

#### 前端 + 後端（Agent Teams）

使用自然語言要求 Claude 建立 Agent Team：

```
建立一個 Agent Team 來產生程式碼，生成 2 個 Teammate：

1. backend-coder — 後端程式碼產生器
   - 設計文件：{arch.md 和 db.md 的內容摘要}
   - 讀取專案 CLAUDE.md 了解架構慣例
   - 讀取設定檔取得技術棧 ID
   - 掃描專案現有程式碼學習風格（POJO、Mapper、Service、Controller 各一個範本）
   - 按架構設計產生所有後端程式碼骨架
   - 風格必須與專案完全一致（package、import 順序、註解、縮排）
   - Service 方法含 TODO 標記待實作邏輯
   - 完成後向 frontend-coder 分享 API 端點清單（URL + Method + 請求/回應格式）
   - 使用繁體中文

2. frontend-coder — 前端程式碼產生器
   - 前端技術棧：{FRONTEND_TECH}
   - 設計文件：{spec.md 中的 API 設計摘要}
   - 掃描專案前端目錄，讀取 2-3 個現有頁面作為風格範本
   - 產生前端頁面（HTML/JSP/Vue）、API 呼叫邏輯、表單驗證、表格展示
   - 風格必須與專案完全一致
   - 完成後向 backend-coder 分享 API 依賴清單
   - 使用繁體中文

重要：兩位 Teammate 負責不同目錄（後端改 src/main/java，前端改 webapp/JSP/JS），不會衝突。

任務依賴：兩位同時開始。
完成後：互相確認 API 契約是否一致（端點 URL、參數、回應格式），
不一致的地方由 backend-coder 為準，frontend-coder 調整。
{dry_run_instruction}

使用 delegate mode，我只協調不寫 code。
所有輸出使用繁體中文。
```

> `{dry_run_instruction}`：若 `--dry-run`，加入「只展示檔案清單和關鍵程式碼片段，不實際建立檔案」。

### 7. 更新 .spec/ 檔案

程式碼產生完成後：

1. 產生 `.spec/{slug}/files.md`：

```markdown
# 程式碼清單

## 新增檔案

| 檔案路徑 | 層級 | 說明 |
|---------|------|------|
| {path} | {Controller/Service/DAO/...} | {說明} |

## 修改檔案

| 檔案路徑 | 修改說明 |
|---------|---------|
```

2. 更新 `README.md` 的 `status: 開發中`
3. 在 `log.md` 追加紀錄

### 8. 回傳結果

```
程式碼產生完成！

📁 產出清單：.spec/{slug}/files.md
📊 統計：N 個後端檔案 {+ M 個前端檔案}

已完成：
  ✅ backend-coder — N 個檔案（POJO/Mapper/Service/Controller）
  {✅ frontend-coder — M 個檔案（JSP/JS/CSS）}
  {✅ API 契約確認 — 一致}

後續可使用：
  • /plan-review   — Agent Teams 3 人審查
  • /plan-close    — 結案並同步 Notion
```

---

## 邊界情況

- **arch.md 不存在**：警告並詢問，允許從 spec.md 或 README.md 直接產生
- **Agent Teams 未啟用**：顯示設定指引
- **--dry-run 模式**：不建立任何檔案，只展示清單和關鍵片段
- **Teammate 失敗**：提供選項：重試 / 跳過 / 終止
- **API 契約不一致**：以 backend-coder 為準，frontend-coder 調整
- **每個工作階段只能一個 Team**：建立新 Team 前確認無殘留 Team
