---
title: Linux 實作、權限與 XSS
categories: [learning, security]
description: 資安第二天：架 Ubuntu 虛擬機、學 Linux 權限、分層除錯 DNS，再看懂 Cookie 和 XSS。
---

架好 Linux 實驗環境，練基本指令和權限，實際解了一個網路問題，最後看懂 Cookie 和 XSS。

## 架好 Linux 實驗環境

在 VirtualBox 裝了 Ubuntu 虛擬機當練習用的實驗室。虛擬機是一個**安全沙盒**，裡面怎麼亂搞都不會影響真實電腦。

![VirtualBox 裡的 Ubuntu-Lab2 虛擬機](/assets/images/learning/day2/virtualbox.png)
*VirtualBox 裡跑起來的 Ubuntu-Lab2*

過程中學到的：

- **以為裝好了其實沒有**：`whoami` 回傳 `ubuntu`，才發現自己還在 Live 試用模式。
- **磁碟加密**：練習機不用加密。磁碟加密保護的是電腦被偷走時的「靜態資料」，對應 CIA 的機密性。

## 你是誰、你在哪、你能做什麼

| 指令 | 作用 |
|---|---|
| `whoami` | 我是誰（目前登入的使用者） |
| `pwd` | 我在哪個資料夾 |
| `id` | 完整身分：uid、gid、所屬群組 |

![ls -l、pwd、id 和 touch 的執行結果](/assets/images/learning/day2/whoami-id.png)
*`id` 裡的 `27(sudo)` 代表這個帳號可以用 sudo*

- **uid**：使用者編號，第一個一般使用者是 1000，root 是 0。
- **群組**是權限的分配單位。`id` 只告訴你「你是誰、屬於哪些群組」，真正能不能做某件事，要等你去做的時候，系統再拿群組和檔案權限來判斷，那才是授權。
- **sudo**（Super User DO）：只在那一行指令借用最高權限，做完立刻變回普通使用者，而且每次都要輸密碼。這就是**最小權限原則**。

其他常用指令：`ls`（`-l` 詳細、`-a` 含隱藏檔）、`cat`、`touch`、`echo`、`chmod`。不會用就打 `指令 --help`。

- **絕對路徑**以 `/` 開頭，從根目錄出發；**相對路徑**從目前位置出發。改系統檔案要用絕對路徑，開頭的 `/` 不能少。
- **管線 `|`** 把左邊指令的輸出接到右邊指令的輸入，像把 C++ 的 `cout` 接到下一個程式的 `cin`。

## 檔案權限（今天重點）

```
d  rwx  r-x  r-x
|   |    |    +-- 其他人 others
|   |    +------- 同群組 group
|   +------------ 擁有者 user
+---------------- 類型（d = 資料夾，- = 檔案）
```

| 權限 | 對檔案 | 對資料夾 |
|---|---|---|
| r 讀（4） | 看內容 | 列出裡面有什麼 |
| w 寫（2） | 修改內容 | 增刪裡面的檔案 |
| x 執行（1） | 當程式執行 | 進入（cd） |

每一組加起來就是一位數字：`rw-r--r--` = 644、`rwxr-xr-x` = 755、`rw-------` = 600。`chmod` 可以用數字一次設好（`chmod 644 檔案`），也可以用符號只改一項（`chmod u+x 檔案`）。

**實作：先鎖住再解鎖**。`chmod 000 test.txt` 拿掉所有權限後，連擁有者自己 `cat` 都會 `Permission denied`；`chmod u+rw` 加回來就能讀了。

![chmod 解鎖後成功讀取 test.txt](/assets/images/learning/day2/chmod-unlock.png)
*被擋下 → 用 chmod 加回讀寫權限 → 權限變成 -rw-------*

Linux 權限「認權限碼，不認交情」，自己的檔案也一樣。

- 機密檔案（密碼檔、SSH 私鑰）標準是 **600**，SSH 私鑰權限太寬鬆時 SSH 會直接拒用。
- 看到重要檔案是 **777** 要警覺。
- 權限設錯是最常見的漏洞之一，進到一台機器後常常先敲 `ls -l` 找「不該開放卻開放」的檔案。突破 `Permission denied` 就是**提權**。

## 網路除錯實戰

`sudo apt update` 失敗，錯誤是「暫時無法解析 security.ubuntu.com」。我用**分層診斷**一層一層找：

1. 看到「解析」兩個字，先懷疑 **DNS**。
2. `ping 8.8.8.8` 成功、`ping google.com` 失敗 → 連線是通的，壞的是「網址翻成 IP」，鎖定 DNS。
3. 改了 Ubuntu 的 `/etc/resolv.conf` 還是不通 → 不是 Ubuntu 內部設定的問題。
4. 判斷問題出在 VirtualBox 的 **NAT 這一層**：DNS 查詢經過它時出了問題。改成**橋接介面卡**，讓虛擬機直接接到家裡的網路後就好了。

![ping google.com 沒反應，ping 8.8.8.8 成功](/assets/images/learning/day2/ping-ip.png)
*ping 網址卡住（按了 Ctrl+C），ping IP 卻通 → 問題在 DNS*

![改成橋接後 apt update 成功下載](/assets/images/learning/day2/apt-update-ok.png)
*改成橋接後，apt update 終於能從 security.ubuntu.com 下載了*

| 指令 | 作用 |
|---|---|
| `ping -c 4 目標` | 測試能不能連到目標（測 4 次） |
| `nslookup 網址` | 測試 DNS 查詢 |
| `cat /etc/resolv.conf` | 看目前用的 DNS 設定 |

## Cookie 與 Session

用 F12 → Application → Cookies 看自己帳號的 Cookie。

- 有一串亂碼的 Cookie 就是網站「記得你已登入」的憑證，刪掉就會被登出。
- 反過來，**別人拿到這串值就能冒充你登入**，不需要密碼，這就是 **Session Hijacking**。所以絕對不要把 Cookie 值給任何人。

![Google 的 Cookie 名稱，大多有 __Secure- 前綴](/assets/images/learning/day2/google-cookies.png)
*Google 的登入 Cookie 大多帶 `__Secure-` 前綴（這裡只截名稱，沒有值）*

- **`__Secure-` 前綴**：只能走 HTTPS 傳送，擋中間人竊聽。
- **HttpOnly**：JavaScript 讀不到這個 Cookie，擋 XSS 偷取。

## XSS 跨站腳本攻擊

駭客把自己的 JavaScript 塞進網站，讓它在**其他訪客的瀏覽器**裡執行，用來偷 Cookie。

根本原因是網站**把使用者輸入的資料當成程式碼執行**。資料和程式碼的界線被打破，是一大類攻擊的共同病根，SQL Injection 也一樣。

| 類型 | 運作方式 | 比喻 |
|---|---|---|
| 儲存型 Stored | 惡意腳本存進資料庫，每個看到的人都中招 | 在公用水井下毒 |
| 反射型 Reflected | 腳本藏在連結裡，騙你點了才執行 | 騙你喝一杯毒飲 |
| DOM 型 | 瀏覽器端的 JavaScript 處理出問題 | （較進階） |

防禦：

1. **輸出轉義**（最核心）：把 `<script>` 這類字元轉成純文字，只顯示不執行。
2. **HttpOnly Cookie**：就算 XSS 成功，也偷不到 Cookie。
3. **CSP**：只允許信任來源的腳本執行。

## 今天的收穫

今天學到了比程式碼更重要的事：處理問題時要先排錯，不要急，一步一步把可能的原因排除掉。

另外，我終於看懂 Cookie 到底是什麼、有什麼作用，也看懂了 Google 是怎麼防止 Cookie 被盜的。還熟悉了很多 Linux 的基本指令。
