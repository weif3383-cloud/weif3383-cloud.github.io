---
title: SQL Injection：從登入繞過到 sqlmap 倒光資料庫
categories: [learning, security]
description: 社課打一個校務系統靶場，從 SQL 基礎、手動 UNION 注入，一路到用 sqlmap 自動化破解，五關全破。
---

今天社課打 SQL Injection，用一個校務系統靶場（Zero2SQL，MariaDB + Docker），總共五關。前兩關手動打、搞懂每個 payload 在幹嘛，後三關用 sqlmap 自動化。這篇把從 SQL 基礎到五關的過程整理起來。

> 以下都是在刻意留了漏洞的練習靶場裡進行，裡面的帳密、flag、資料全是靶場內建的假資料。這些技術只能用在自己的、或明確授權的系統上——未經授權對他人網站做注入測試，在台灣會觸犯《刑法》妨害電腦使用罪，不論有沒有得手。

## SQL 與資料庫基礎

資料庫可以想成「一疊 Excel 表格」。三個核心名詞：

- **表格（table）**：一個主題的資料集，例如 `courses`、`users`
- **欄位（column）**：直行，是屬性，例如 `name`、`password`
- **資料列（row）**：橫列，是一筆紀錄

跟資料庫溝通用 **SQL**，最常見的查詢是：

```sql
SELECT 欄位 FROM 表格 WHERE 條件;
```

SQL 注入的根本病因，跟 XSS 是同一個：

> **網站把「使用者輸入的資料」當成「SQL 指令」來執行了。**

本該是純資料的輸入（帳號、關鍵字），被拼進 SQL 後變成可執行的指令——「資料 vs 指令的界線沒守住」。

## 注入三件套

手動注入的 payload 幾乎都由三部分組成：

1. **閉合引號**：用 `'` 把網站原本的左引號關掉（數字型則不用引號）
2. **payload**：`OR 1=1`、`UNION SELECT ...` 等
3. **註解收尾 `-- `**：把網站 SQL 剩下的尾巴剪掉，避免語法錯誤

踩到的坑：MySQL/MariaDB 的 `--` 後面**一定要有空格**才算註解（不然會跟「減負號」搞混）。保險寫法是 `-- -`。

## 三大注入流派

差別只在「查詢結果看不看得到」，跟 SQL 怎麼寫無關：

| 流派 | 網站會顯示結果嗎 | 用什麼招 |
|---|---|---|
| UNION 注入 | 會（直接看到資料） | `UNION SELECT` 偷接別張表 |
| 布林盲注 | 只有成功/失敗差異 | `substring` + 比對，逐字猜 |
| 時間盲注 | 什麼差異都沒有 | `IF(條件, SLEEP(n), 0)`，用延遲當訊號 |

「盲」不是資料庫沒回傳，而是**程式拿到資料卻不顯示給你**。

## LV1 — 登入繞過

後端大概是 `SELECT ... FROM users WHERE username='[帳號]' AND password='[密碼]'`。帳號欄輸入：

```
' OR 1=1 -- 
```

`username=''` 閉合 → `OR 1=1` 永真 → `-- ` 剪掉密碼檢查 → 直接登入成 admin。

*不知道任何帳密，用 OR 1=1 直接登入成 admin*

## LV2 — UNION 注入 + information_schema

這關是數字型注入（`WHERE id=[編號]`，不用引號）。完整偵查流程：

1. 數欄位：`1 ORDER BY 4-- -` 正常 → 4 欄
2. 找顯示位置：`0 UNION SELECT 1,2,3,4-- -` → 四格全可顯示
3. 列表格：`information_schema.tables` 撈出有哪些表
4. 列欄位：`information_schema.columns` 撈出目標表的欄位
5. 撈資料：`UNION SELECT filename,content,3,4 FROM secret_files-- -`

```
0 UNION SELECT group_concat(table_name),2,3,4 FROM information_schema.tables WHERE table_schema=database()-- -
```

*information_schema 是資料庫「自己的目錄」，記著有哪些表、哪些欄——它是普通 SELECT，所以能塞進 UNION*

學到最重要的一點：**一旦有注入點，透過子查詢能讀到整個資料庫的任何一張表**，不限於你注入的那張。開頭的 `0` 只是佔位值，讓前半段查空、畫面只剩偷來的資料。

## LV3~LV5 — sqlmap 自動化

後三關改用 **sqlmap**（自動化注入工具）。GET 請求直接給網址，POST 表單用 `--forms`：

```bash
# GET
sqlmap -u "http://靶機/library.php?q=a" --batch --dump
# POST 表單
sqlmap -u "http://靶機/feedback.php" --forms --batch --dump
```

- `--batch`：自動用預設答案回答所有問題
- `--dump`：把資料表內容倒出來

sqlmap 分兩階段：先偵測（怎麼注入），再 `--dump`（偷資料）——跟手動流程一樣，只是自動化。它會自動列舉資料庫、表、欄位，連「猜不到的隱藏表」都挖出來。

*sqlmap 自動找到 boolean-based、error-based、time-based 三種注入——全是手動學過的技術的自動版*

| 關卡 | 技術 | 方式 |
|---|---|---|
| LV1 | 登入繞過 `OR 1=1` | 手動 |
| LV2 | UNION + information_schema | 手動 |
| LV3 | error-based 盲注 | sqlmap |
| LV4 | boolean 盲注 | sqlmap |
| LV5 | time-based 盲注（POST 表單） | sqlmap `--forms` |

LV5 特別有意思：表單看起來是「意見信箱」，但後端其實是拿 email 去查訂閱者（existence check），結果不顯示 → 盲注。而注入點只是查 email，flag 卻在一張完全不同的 `principal_secrets` 表——再次印證「有注入點 = 整個資料庫都讀得到」。

## 防禦：怎麼修好它

| 防禦 | 說明 |
|---|---|
| **參數化查詢（Prepared Statement）** | 最根本：讓使用者輸入永遠只當資料、無法變成 SQL 指令 |
| 最小權限 | 網站的資料庫帳號別用 root，被注入時損害有限 |
| 關閉詳細錯誤訊息 | 擋 error-based、避免洩漏資料庫結構 |

病根是「資料/指令混在一起」，**參數化查詢就是把它們永久分開**，其他都是輔助。

## 心得

今天學到了 sqlmap 這個工具，以及它背後的運作原理，讓我了解到用這個工具能做到什麼的同時，還能知道它背後的原理，而不是亂用。

社課的課程十分好，他寫的環境和練習我都很愛，尤其是意見回饋那一題——校長根本沒有想看郵件，所以直接用 `SELECT` 查 XD。

