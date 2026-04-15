---
name: plan-browse
description: 瀏覽與探索已有的 .spec/ 規劃文件，支援深度閱讀、跨任務比較、模式搜尋。當使用者提到「plan-browse」、「瀏覽規劃」、「看一下之前的設計」、「哪些任務有類似的」、「browse」、「查看規劃」時觸發此 Skill。
---

# plan-browse — 規劃瀏覽器（零 Notion 呼叫）

深度瀏覽 `.spec/` 目錄中的規劃文件。不只是列出任務（那是 `/plan-status` 的工作），而是**讀取、理解、比較、搜尋**設計內容。

---

## 使用方式

```
/plan-browse                         # 互動式瀏覽（列出所有規劃，選擇後深入）
/plan-browse <slug>                  # 深度閱讀指定任務的所有設計文件
/plan-browse --compare <slug1> <slug2>  # 比較兩個任務的設計
/plan-browse --search <關鍵字>        # 跨任務搜尋設計內容
/plan-browse --patterns              # 分析跨任務的共通模式
/plan-browse --timeline              # 按時間軸瀏覽規劃演進
```

---

## 與 plan-status 的差異

| | `/plan-status` | `/plan-browse` |
|--|----------------|----------------|
| 目的 | 任務管理（狀態、暫停/恢復） | 設計內容探索 |
| 深度 | 列出名稱、狀態、分支 | 讀取並摘要設計文件內容 |
| 比較 | 不支援 | 支援跨任務比較 |
| 搜尋 | 不支援 | 支援關鍵字搜尋 |
| 模式 | 不支援 | 分析共通設計模式 |

---

## 流程

### 模式 1：互動式瀏覽（無參數）

1. 掃描 `.spec/` 目錄，讀取每個任務的 README.md 和所有設計文件
2. 呈現豐富的總覽（不只是名稱和狀態）：

```
📚 規劃瀏覽器

找到 {N} 個規劃：

### 1. 🔧 推播標籤查詢（push-tag-query）
   狀態：架構設計 | 分支：feature/push-tag-query
   ┌──────────────────────────────────────────┐
   │ 摘要：新增標籤查詢 API，支援多條件篩選推播對象 │
   │ API：3 個端點（GET/POST/DELETE）           │
   │ DB：2 個新表 + 3 個索引                    │
   │ 架構：Strategy 模式處理不同查詢類型         │
   └──────────────────────────────────────────┘

### 2. 🐞 SSO 登入異常（sso-login-fix）
   狀態：調查中 | 分支：hotfix/sso-login-fix
   ┌──────────────────────────────────────────┐
   │ 摘要：Spring Security 過濾鏈順序導致 SSO 失效 │
   │ 根因：FilterChainProxy 載入順序錯誤          │
   │ 影響範圍：所有 SSO 登入使用者                │
   └──────────────────────────────────────────┘

輸入編號深入閱讀，或使用：
  • --compare 1 2  比較兩個規劃
  • --search <關鍵字>  搜尋設計內容
```

3. 使用者選擇後，進入深度閱讀模式

### 模式 2：深度閱讀（指定 slug）

讀取 `.spec/{slug}/` 下**所有**設計文件，產出結構化摘要：

```
📖 深度閱讀：推播標籤查詢（push-tag-query）

═══════════════════════════════════════════════
  📋 需求（README.md）
═══════════════════════════════════════════════
  • 目標：{一句話摘要}
  • 類型：Feature
  • 建立日期：2026-03-16
  • Notion：{URL}

═══════════════════════════════════════════════
  📐 技術規格（spec.md）
═══════════════════════════════════════════════
  範圍：
    ✅ In：{列出}
    ❌ Out：{列出}

  API 端點：
    ┌──────────┬────────────────────┬──────────┐
    │ 方法     │ 路徑               │ 說明     │
    ├──────────┼────────────────────┼──────────┤
    │ GET      │ /api/push/tags     │ 查詢標籤 │
    │ POST     │ /api/push/tags     │ 新增標籤 │
    │ DELETE   │ /api/push/tags/{id}│ 刪除標籤 │
    └──────────┴────────────────────┴──────────┘

  業務規則：{N} 條
  判斷：DB_REQUIRED=true, FRONTEND_REQUIRED=true

═══════════════════════════════════════════════
  🗄️ 資料庫設計（db.md）
═══════════════════════════════════════════════
  表清單：
    • push_tags（主表，{N} 個欄位）
    • push_tag_conditions（條件表，{N} 個欄位）

  關聯：
    push_tags 1──N push_tag_conditions

  索引：{N} 個

═══════════════════════════════════════════════
  🏗️ 架構設計（arch.md）
═══════════════════════════════════════════════
  設計模式：Strategy
  類別清單：{N} 個
  分層：Controller → Service → Repository → Entity

═══════════════════════════════════════════════
  📝 開發日誌（log.md）
═══════════════════════════════════════════════
  • [2026-03-16] plan-spec 完成
  • [2026-03-16] plan-db 完成
  • [2026-03-17] plan-arch 完成
```

深度閱讀後，提供互動選項：

```
後續操作：
  • 「這個 API 設計為什麼選 GET 而不是 POST？」— 進入探索討論
  • 「跟 XXX 比較一下」— 比較模式
  • 「繼續規劃」— 回到 /plan-spec 或 /plan-arch
```

### 模式 3：比較模式（--compare）

讀取兩個任務的所有設計文件，逐層比較：

```
⚖️  規劃比較：push-tag-query vs subscription-stats

┌─────────────────┬──────────────────┬──────────────────┐
│ 面向            │ push-tag-query   │ subscription-stats│
├─────────────────┼──────────────────┼──────────────────┤
│ 類型            │ Feature          │ Feature           │
│ API 數量        │ 3                │ 2                 │
│ DB 表數量       │ 2                │ 0（查詢既有表）    │
│ 設計模式        │ Strategy         │ Template Method   │
│ 前端需求        │ JSP              │ JSP               │
│ 複雜度          │ 中               │ 低                │
└─────────────────┴──────────────────┴──────────────────┘

共通點：
  • 都使用 Spring Data JPA Repository
  • 都有分頁查詢需求
  • Controller 都在 com.intumit.action 套件

差異點：
  • push-tag-query 需要新建表，subscription-stats 只查詢
  • push-tag-query 用 Strategy 模式，因為查詢條件多樣
  • subscription-stats 較簡單，可直接參考既有 Controller 模式

可復用的設計：
  • 分頁查詢的 DTO 結構可共用
  • Repository 的自訂查詢寫法一致
```

### 模式 4：搜尋模式（--search）

跨所有 `.spec/` 目錄搜尋設計文件內容：

```bash
# 在所有 .spec/ 下的 .md 檔案中搜尋
grep -r "<關鍵字>" .spec/ --include="*.md" --include="*.sql"
```

格式化搜尋結果：

```
🔍 搜尋「RabbitMQ」— 找到 {N} 筆

1. push-tag-query/spec.md:42
   「...透過 RabbitMQ 發送推播訊息到 LineBC...」

2. push-tag-query/arch.md:78
   「...RabbitMqPushAdapter 實作非同步推播...」

3. subscription-notify/spec.md:15
   「...訂閱通知使用 RabbitMQ fanout exchange...」

相關任務：push-tag-query, subscription-notify
共通模式：兩者都使用 Spring AMQP + RabbitTemplate
```

### 模式 5：模式分析（--patterns）

分析所有規劃中的共通設計模式：

```
🔬 跨任務設計模式分析

掃描了 {N} 個規劃，發現以下模式：

## API 設計模式
  • RESTful CRUD：出現 {N} 次（push-tag-query, ...）
  • 查詢 + 匯出：出現 {N} 次（subscription-stats, ...）

## DB 設計模式
  • 主表 + 明細表：出現 {N} 次
  • 使用 NVARCHAR：所有任務一致
  • 索引命名：idx_{table}_{column}

## 架構模式
  • Strategy：{N} 次（多條件查詢場景）
  • Template Method：{N} 次（固定流程、可變步驟）
  • Adapter：{N} 次（外部 API 整合）

## 可復用元件
  • 分頁 DTO（PageRequest/PageResponse）— 已在 3 個任務出現
  • 通用 API Response 格式 — 所有任務一致
  • Solr 查詢工具類 — 2 個任務使用
```

### 模式 6：時間軸（--timeline）

按時間順序展示規劃演進：

```
📅 規劃時間軸

2026-03
├── 03-10 ✅ subscription-stats（已完成）
│         訂閱統計去重複計算
├── 03-15 ⏸️  data-export（暫停中）
│         資料匯出功能
├── 03-16 🔧 push-tag-query（架構設計）
│         推播標籤查詢
│         ├── 03-16 spec.md 完成
│         ├── 03-16 db.md 完成
│         └── 03-17 arch.md 完成
└── 03-17 🐞 sso-login-fix（調查中）
          SSO 登入異常
```

---

## 邊界情況

- **`.spec/` 不存在**：提示「還沒有任何規劃，使用 `/plan-start` 建立第一個」
- **`.spec/` 為空**：同上
- **指定的 slug 不存在**：列出可用的 slug 供選擇
- **設計文件不完整**（例如只有 README.md 沒有 spec.md）：顯示已有的文件，標記缺少的
- **搜尋無結果**：提示「未找到匹配結果」，建議更換關鍵字或使用 `/plan-explore` 探索
- **只有一個規劃**：比較模式不可用，提示改用深度閱讀
- **README.md frontmatter 損壞**：顯示警告，嘗試從檔案內容推斷資訊

---

## 與其他指令的銜接

瀏覽後使用者可能想：

| 意圖 | 建議指令 |
|------|---------|
| 「這個設計有問題，想討論一下」 | `/plan-explore <slug>` |
| 「想修改這個規格書」 | `/plan-spec`（會進入編輯迴圈） |
| 「這個可以開始寫了」 | `/plan-build` |
| 「想建立類似的新任務」 | `/plan-start` |
| 「暫停這個任務」 | `/plan-status --park <slug>` |

---

## Gotchas

- **不要修改設計文件**：plan-browse 是唯讀模式。如果使用者想修改，引導到對應的 plan-* 指令或 `/plan-explore`
- **大量規劃時的效能**：如果 `.spec/` 下有超過 20 個目錄，互動式瀏覽只顯示最近 10 個（按修改時間排序），並提示使用 `--search` 或 `--timeline` 瀏覽全部
- **摘要品質**：深度閱讀的摘要應忠於原文，不要添加原文沒有的推測
- **比較時保持中立**：不自動判定哪個設計「更好」，除非使用者問
