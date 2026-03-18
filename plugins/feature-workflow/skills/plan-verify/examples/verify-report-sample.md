# 範例：verify.md 驗證報告

以下展示 `/plan-verify` 的理想產出格式，涵蓋四種狀態（PASS / FAIL / SKIP / MANUAL）。

---

# 驗證報告

## 摘要

| 項目 | 值 |
|------|-----|
| 驗證日期 | 2026-03-18 |
| 環境 | localhost:8080 |
| 模式 | 完整 |
| 驗證工具 | chrome-devtools-mcp v0.20.1 |

## 統計

| 狀態 | 數量 |
|------|------|
| ✅ PASS | 3 |
| ❌ FAIL | 1 |
| ⏭️ SKIP | 1 |
| 👤 MANUAL | 1 |

## 驗證結果

### [1] ✅ 可依日期範圍查詢推播統計

- **類型**：API
- **驗證**：`GET /ap/pushTagQuery/list?startDate=2026-01-01&endDate=2026-03-18&pageNum=1&pageSize=20`
- **結果**：HTTP 200, 回傳 15 筆資料，格式正確
- **證據**：
  ```json
  {
    "code": "0000",
    "data": { "list": [...], "total": 15, "pageNum": 1 }
  }
  ```

### [2] ✅ 日期範圍超過 90 天回傳錯誤

- **類型**：API
- **驗證**：`GET /ap/pushTagQuery/list?startDate=2025-01-01&endDate=2026-03-18`
- **結果**：HTTP 400, 回傳 `DATE_RANGE_EXCEEDED`
- **證據**：
  ```json
  { "code": "DATE_RANGE_EXCEEDED", "message": "查詢區間不可超過 90 天" }
  ```

### [3] ✅ 支援分頁顯示

- **類型**：UI
- **驗證**：
  1. `take_snapshot()` — 確認表格顯示 20 筆
  2. `click({ selector: ".pagination .next" })` — 點擊下一頁
  3. `wait_for({ text: "第 2 頁" })` — 等待頁面更新
  4. `take_snapshot()` — 確認表格資料已更新
- **證據**：第一頁顯示 id 1-20，第二頁顯示 id 21-40
- **截圖**：screenshots/verify-3-pagination.png

### [4] ❌ 匯出 Excel 功能

- **類型**：UI
- **驗證**：
  1. `take_snapshot()` — 尋找匯出按鈕
  2. 未找到匹配的匯出按鈕元素
- **失敗原因**：snap 中未找到匯出按鈕（`#exportBtn` 不存在於無障礙樹中）。頁面上可能使用了不同的 selector 或按鈕尚未實作。
- **預期**：頁面應有「匯出 Excel」按鈕，點擊後下載 .xlsx 檔案
- **實際**：無障礙樹中無任何包含「匯出」文字的可點擊元素
- **截圖**：screenshots/verify-4-export-missing.png

### [5] ⏭️ 標籤模糊搜尋即時反應

- **類型**：UI
- **跳過原因**：前端頁面尚未實作搜尋框（spec 中 `FRONTEND_REQUIRED: false`，本期僅做 API）

### [6] 👤 統計數據與 LINE 後台一致

- **類型**：資料驗證
- **說明**：需人工登入 LINE Official Account Manager 比對推播統計數據
- **手動驗證步驟**：
  1. 登入 LINE Official Account Manager (manager.line.biz)
  2. 進入「分析」>「訊息」
  3. 選擇與 API 相同的日期範圍（2026-03-01 ~ 2026-03-18）
  4. 比對「發送數」、「開封數」欄位是否與系統查詢結果一致
  5. 允許 ±1% 的誤差（LINE 後台統計有延遲）
- **截圖**：screenshots/verify-6-manual-guide.png
