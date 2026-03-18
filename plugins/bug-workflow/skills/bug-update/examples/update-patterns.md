# 範例：bug-update 各種更新模式

以下展示 `/bug-update` 的常見更新模式，供 Claude 判斷內容類型和格式化方式時參考。

---

## 1. Log 貼入

使用者輸入：
```
/bug-update
ERROR 2026-03-18 14:32:15 [push-pool-3] c.x.linebc.message.PushService - 推播失敗
java.lang.NullPointerException: null
    at com.xtremeapp.linebc.message.PushService.sendMulticast(PushService.java:235)
    at com.xtremeapp.linebc.schedule.PushBySchedule.execute(PushBySchedule.java:87)
```

寫入 Notion（調查過程 > 關鍵 Log）：

```
[14:35] AP1 catalina.out：

​```
ERROR 2026-03-18 14:32:15 [push-pool-3] c.x.linebc.message.PushService - 推播失敗
java.lang.NullPointerException: null
    at com.xtremeapp.linebc.message.PushService.sendMulticast(PushService.java:235)
    at com.xtremeapp.linebc.schedule.PushBySchedule.execute(PushBySchedule.java:87)
​```
```

> 注意：時間戳 `[14:35]` 用 `date +%H:%M` 取本地時間，不用 UTC。

---

## 2. SQL 查詢記錄

使用者輸入：
```
/bug-update
SELECT TOP 10 msg_id, push_status, error_msg, create_date
FROM msg_main
WHERE push_status = 'F' AND create_date > '2026-03-18'
ORDER BY create_date DESC

-- 結果：發現 3 筆 error_msg = 'token expired'
```

寫入 Notion（調查過程 > 相關 SQL 查詢）：

```
[14:40] 查詢失敗推播記錄：

​```sql
SELECT TOP 10 msg_id, push_status, error_msg, create_date
FROM msg_main
WHERE push_status = 'F' AND create_date > '2026-03-18'
ORDER BY create_date DESC
​```

結果：3 筆 `error_msg = 'token expired'`，最早發生於 14:15。
```

---

## 3. 初步判斷與根因推理

使用者輸入：
```
/bug-update 初步判斷：
1. PushService.sendMulticast 在第 235 行呼叫 accessToken.getValue()
2. accessToken 是從 Param.getLineAccessToken() 取得
3. AccessTokenSchedule 每 15 天更新一次，上次更新是 3/3
4. 但 LINE 平台在 3/17 強制更換了 token（LINE 公告）
5. 推測是 token 已被 LINE 平台廢止但排程尚未更新
```

寫入 Notion（調查過程 > 初步判斷）：

```
[14:50] 初步判斷：

1. `PushService.sendMulticast` 在第 235 行呼叫 `accessToken.getValue()`
2. `accessToken` 從 `Param.getLineAccessToken()` 取得
3. `AccessTokenSchedule` 每 15 天更新，上次更新 3/3
4. LINE 平台 3/17 強制更換 token（LINE 公告）
5. **推測**：token 已被 LINE 平台廢止，但排程尚未觸發更新
```

---

## 4. Reopen 復發紀錄格式

使用者輸入：
```
/bug-update reopen https://www.notion.so/abe41af9...
正式環境仍出現相同的 NullPointerException，但只有 AP2 有問題
```

Notion 頁面更新：

**狀態**：已完成 → 進行中

**新增「復發紀錄」區塊**（插入在「驗證」區塊之前）：

```
---

## 🔄 復發紀錄

### [2026-03-18 復發] 正式環境仍出現相同的 NullPointerException，但只有 AP2 有問題
- **復發環境**：正式環境 AP2
- **復發現象**：與上次相同的 NullPointerException at PushService.java:235
- **前次修復為何無效**：

---
```

**取消驗證勾選**：將已勾選的 `- [x]` 改回 `- [ ]`

> 注意：多次復發時，每次在「復發紀錄」區塊下新增子區塊，不覆蓋之前的。
