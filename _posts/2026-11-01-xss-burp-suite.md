---
title: XSS 實戰與 Burp Suite 入門
categories: [learning, security]
description: 在 PortSwigger 靶場打通第一關 XSS，搞懂同源政策和偷 Cookie 的攻擊鏈，再用 Burp Suite 解剖自己的 Cookie。
---

在 PortSwigger 的免費靶場親手打了 XSS，接著弄懂攻擊鏈和同源政策，最後裝了 Burp Suite 觀察自己的 Cookie。

## 反射型 XSS：打通第一關

用 PortSwigger Web Security Academy 打通第一關「Reflected XSS into HTML context with nothing encoded」。比彈窗本身更重要的是**偵察流程**：

1. **找入口**：搜尋框是使用者輸入點。
2. **無害試探**：先送 `hello`，確認輸入會反射回頁面和網址（`?search=hello`）。
3. **驗證漏洞**：送探針 `<test>`，看原始碼是 `0 search results for '<test>'`，`<` 沒有被轉成 `&lt;`。
4. **送出 payload**：`<script>alert(1)</script>` → 彈窗跳出 → Solved。

關鍵是**轉義**：正經網站會把 `<` 變成 `&lt;`，`<script>` 就只會被顯示成文字、不會執行。真實網站不會把「有沒有防護」寫在標題上，所以要主動送探針、**看原始碼**驗證，因為有沒有轉義在畫面上可能看起來一樣。

## 從 alert 到偷 Cookie

`alert(1)` 只證明腳本能執行。要偷 Cookie 還得用 `document.cookie` 讀出來，再**傳到駭客的伺服器**。腳本是在受害者的瀏覽器裡跑的，駭客不在那台電腦前，沒傳出去就等於白偷。

反射型的完整攻擊鏈：

1. 做一個有毒的網址，網域是**目標真網站**，script 藏在參數裡，例如 `bank.com/?search=<script>...</script>`。
2. 用釣魚手法騙受害者點。
3. script 在受害者的瀏覽器、以 `bank.com` 的身分執行。
4. 讀出 Cookie，傳到駭客的伺服器。
5. 駭客拿 Cookie 冒充他登入（Session Hijacking）。

真實攻擊還會把 payload 藏起來，例如網址編碼、短網址，或改用網址看起來乾淨的儲存型 XSS。

## 同源政策：Web 安全的地基

**一個網站的 JavaScript 只能讀它自己網域的 Cookie**。所以釣魚網站 `evil.com` 讀不到你 Google 或銀行的 Cookie。

XSS 可怕在它**繞過**了這道牆：script 被塞進目標網站本身，瀏覽器以為是網站自己的程式，就讓它讀該網站的 Cookie。就像不是翻牆進鄰居家，而是騙鄰居把你的程式帶進去，於是能用鄰居家的鑰匙。

所以一次 XSS 只能偷**那一個網站**的 Cookie。要偷銀行的，就得在銀行網站本身找到 XSS。

- **釣魚**：做假登入頁，騙你親手打帳密。
- **XSS**：在真網站注入 script，偷那個真網站的 Cookie。

## 通用病根：資料和指令的界線

| 攻擊 | 「資料」是什麼 | 被當成什麼執行 |
|---|---|---|
| XSS | 使用者輸入的文字 | HTML / JavaScript |
| SQL Injection | 使用者輸入的文字 | 資料庫查詢指令 |
| 命令注入 | 使用者輸入的文字 | 作業系統指令 |
| 路徑穿越 | 使用者輸入的檔名 | 檔案系統路徑（`../../`） |

全部都是同一個病：系統沒分清楚「使用者給的資料」和「自己的指令」。防禦也是同一招，**永遠把輸入當資料處理**：XSS 做輸出轉義、SQL 用參數化查詢、命令注入不要把輸入直接丟給系統指令。

**URL 編碼**也是同一件事：網址裡的 `? & / = #` 有特殊功能，資料裡出現時要編成 `%` 加十六進位（`%3C` = `<`）。它也能被拿來繞過：過濾器在解碼前檢查 `%3Cscript%3E` 看不懂而放行，伺服器解碼後又變回 `<script>`。關鍵在「檢查」和「使用」的時機差。

**出站過濾**是我從攻擊反推出來的防禦：一般防火牆擋外面進來的連線，如果連裡面出去的也擋，就算 XSS 讀到 Cookie 也傳不出去。這是最後一道防線，不是根治，和輸出轉義、HttpOnly、CSP 一起構成縱深防禦。

## 儲存型 XSS 與卡關

在部落格文章的留言內容注入 `<script>alert(1)</script>`，重新載入文章就跳出彈窗，確認有儲存型 XSS。script 被存進網站，每個人打開這篇文章都會執行一次，而且畫面上什麼都看不到，比反射型更隱蔽。

接著想打「偷 Cookie」那關卻卡住了：標準解法要用 Burp Collaborator 當接收端，而靶場的防火牆擋掉了連到任意外部網站的連線。這類資料外洩題的正統做法就是用 Burp Suite，剛好帶到下一個主題。

## Burp Suite 入門

Burp Suite 站在瀏覽器和網站中間當**中間人**，把每個請求攔下來讓你**看、改、重送**。網站常常只在前端檢查輸入，或你想改的東西根本沒有輸入框（Cookie、隱藏欄位），用 Burp 就能繞過前端，直接跟伺服器對話。

攔截流程（Proxy → Intercept）：

- **Intercept off**：流量正常通過，但 Burp 會記錄在 HTTP history。
- **Intercept on**：每個請求都被扣住，按 Forward 才放行，按 Drop 就丟掉。網頁一直轉圈是正常的，不是當機。

攔到的請求裡有 `Host`、`User-Agent`、`Accept-Language` 等 header。平常只是打個網址，背後其實送出了語言、作業系統、瀏覽器版本這麼多資訊。

## 用 Burp 解剖自己的 Cookie

連上一個已登入的網站，一個請求就夾帶了 **25 個 Cookie**：

- **登入身分**：`.AspNetCore.Cookies`，偷到就能冒充登入，最值錢。
- **追蹤分析**（佔多數）：`_pk_id`、`_gcl_au`（Google Ads）、`visitor_id` 等，這些就是網站要跳 Cookie 同意視窗的原因。
- **功能性**：`AWSALBAPP`（AWS 負載平衡）。

登入憑證還被**切成三塊**：`.AspNetCore.Cookies`、`…C1`、`…C2`。因為單一 Cookie 大約只能放 4KB，憑證太長就分塊，網站收到再拼回來。駭客必須把每一塊都偷齊才能用，加上 HttpOnly 讓 JavaScript 讀不到，真實世界偷 Cookie 沒有想像中簡單。

## 今天的收穫

1. 親手完成第一次 XSS 攻擊（反射型 Solved），也自己驗證出儲存型 XSS。
2. 學習到進攻流程：偵察 → 無害試探 → 看原始碼驗證 → 送 payload。
3. 同源政策是 Web 安全的地基，XSS 的本質是借用目標網站自己的身分。
4. 「資料和指令的界線」貫穿 XSS、SQLi、命令注入和 URL 編碼。
5. 每學一個攻擊就反推防禦，自己推出了出站過濾。
6. 裝好 Burp Suite，第一次當中間人，解剖了自己的 25 個 Cookie。
