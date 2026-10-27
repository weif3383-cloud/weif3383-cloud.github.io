---
title: 掃地機器人第一版：能跑就行 XD
categories: project
# cover: /assets/images/projects/robot-vacuum/cover.jpg   # 封面圖還沒決定，先用標題第一個字
description: 這是用來測試的初號機，目前只有跑的功能。
gallery:
  - image: /assets/images/projects/robot-vacuum/01.jpg
    caption: 組裝完成，用手機網頁的搖桿遙控
  - image: /assets/images/projects/robot-vacuum/02.jpg
    caption: 在書桌前焊接、接線
---

這是掃地機器人的初號機，主要用來測試底盤和馬達，目前只有「跑」的功能：手機連上車子的 WiFi，就能用網頁上的搖桿遙控它前進、後退和轉彎。

## 怎麼做的

### 材料

| 零件 | 數量 | 用途 |
|------|------|------|
| NodeMCU-32S（ESP32） | 1 | 主控板，自己開 WiFi 熱點、跑遙控網頁 |
| L298N 馬達驅動板 | 1 | 控制兩顆馬達的轉向和速度 |
| TT 減速馬達 + 輪子 | 2 | 左右兩輪，差速轉彎 |
| 圓形底盤 | 1 | 放所有零件 |
| 電池盒、杜邦線 | 若干 | 供電與接線 |

### 底盤設計

底盤是直徑 180 mm 的圓形，底板厚 3 mm、外圈高 8 mm。電池盒放在輪軸前面，讓重心偏前、車子不會往後翹；L298N 放在兩顆馬達中間；車頭的外圈留了缺口，之後要裝超音波感測器。

![第一版底盤平面圖（俯視）](/assets/images/projects/robot-vacuum/chassis-v1.png)
*第一版底盤平面圖（俯視）：藍色是定位牆，橘色是零件的對齊記號，兩側缺口是輪槽。*

### 程式碼

手機連上 WiFi「RobotCar」後，打開 `http://192.168.4.1` 就會出現搖桿頁面。超過 0.5 秒沒收到手機的指令（斷線、切換 App、手機鎖螢幕），車子會自動停下來。

<details markdown="1">
<summary>點我展開完整程式碼（Arduino / ESP32）</summary>

{% raw %}
```cpp
// 遙控小車 v1 — 手機網頁搖桿（ESP32 自己開 WiFi 熱點）
// 板子：NodeMCU-32S（Arduino-ESP32 核心 3.x），不需要另外安裝函式庫
// 用法：燒錄後，手機連 WiFi「RobotCar」（密碼 12345678），瀏覽器打開 http://192.168.4.1
// 安全設計：超過 0.5 秒沒收到手機指令（斷線、切換 App、手機鎖螢幕）就自動停車
#include <Arduino.h>
#include <WiFi.h>
#include <WebServer.h>

// ---- WiFi 熱點 ----
const char* AP_SSID = "RobotCar";
const char* AP_PASS = "12345678";   // 至少 8 碼，可以自己改

// ---- 腳位與方向（和 car_test v2.3 相同）----
const int L_IN1 = 32, L_IN2 = 33;   // 左輪：L298N IN3 / IN4
const int R_IN1 = 26, R_IN2 = 27;   // 右輪：L298N IN1 / IN2
const bool L_INVERT = true;
const bool R_INVERT = true;
const int PWM_FREQ = 1000, PWM_RES = 8;

// ---- 可調參數 ----
int MIN_PWM = 60;    // 輪子剛好能轉動的 PWM；搖桿輕推時從這裡起跳（車子推不動就調高）
int BAL = 0;         // 直線校正：填 car_test 測出來的 bal 值
const unsigned long CMD_TIMEOUT_MS = 500;

WebServer server(80);
unsigned long lastCmd = 0;
bool moving = false;
int maxPwm = 200;

// s：-255 ~ 255，負值為反轉
void motor(int inA, int inB, int s) {
  s = constrain(s, -255, 255);
  if (s > 0)      { ledcWrite(inA, s); ledcWrite(inB, 0); }
  else if (s < 0) { ledcWrite(inA, 0); ledcWrite(inB, -s); }
  else            { ledcWrite(inA, 0); ledcWrite(inB, 0); }
}

void drive(int left, int right) {
  motor(L_IN1, L_IN2, L_INVERT ? -left : left);
  motor(R_IN1, R_IN2, R_INVERT ? -right : right);
}

// 搖桿量 -1 ~ 1 轉成 PWM；中間有小死區，非零時從 MIN_PWM 起跳
int toPwm(float v) {
  if (fabs(v) < 0.06) return 0;
  int p = MIN_PWM + (int)((maxPwm - MIN_PWM) * fabs(v));
  return v > 0 ? p : -p;
}

const char PAGE[] PROGMEM = R"HTML(<!doctype html>
<html lang="zh-Hant"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no">
<title>小車遙控</title>
<style>
html,body{margin:0;height:100%;background:#12181b;color:#e8eef0;font-family:system-ui,-apple-system,"Noto Sans TC",sans-serif;
  touch-action:none;user-select:none;-webkit-user-select:none;overflow:hidden}
.wrap{display:flex;flex-direction:column;align-items:center;justify-content:space-around;height:100%;padding:16px;box-sizing:border-box}
#st{font-size:15px;color:#8fa3a8}
#st.ok{color:#5fd3a0}#st.bad{color:#ff7a6b}
#pad{width:min(82vw,320px);height:min(82vw,320px);border-radius:50%;background:#1d2a2f;border:2px solid #2f4a52;position:relative}
#pad:before,#pad:after{content:"";position:absolute;background:#2f4a52}
#pad:before{left:50%;top:8%;bottom:8%;width:2px;margin-left:-1px}
#pad:after{top:50%;left:8%;right:8%;height:2px;margin-top:-1px}
#knob{width:36%;height:36%;border-radius:50%;background:#2fa4a9;position:absolute;box-shadow:0 4px 14px #000a;z-index:1}
.row{width:100%;max-width:360px;font-size:15px}
input[type=range]{width:100%;height:32px}
#stop{width:100%;max-width:360px;padding:18px;font-size:20px;border:0;border-radius:14px;background:#c0392b;color:#fff}
</style></head><body><div class="wrap">
<div id="st">連線中…</div>
<div id="pad"><div id="knob"></div></div>
<div class="row">最高速度 <b id="sv">200</b><input id="spd" type="range" min="80" max="255" value="200"></div>
<button id="stop">停止</button>
</div>
<script>
const pad=document.getElementById('pad'),knob=document.getElementById('knob'),st=document.getElementById('st'),
      spd=document.getElementById('spd'),sv=document.getElementById('sv');
let x=0,y=0,active=false,busy=false,zeroSent=true,pid=null,lastOk=0;
function setKnob(){const r=pad.clientWidth/2,k=knob.clientWidth/2;
  knob.style.left=(r-k+x*(r-k))+'px';knob.style.top=(r-k-y*(r-k))+'px';}
function move(e){const b=pad.getBoundingClientRect(),r=b.width/2;
  let dx=(e.clientX-b.left-r)/r,dy=-(e.clientY-b.top-r)/r;const d=Math.hypot(dx,dy);
  if(d>1){dx/=d;dy/=d;}x=dx;y=dy;setKnob();}
pad.addEventListener('pointerdown',e=>{pid=e.pointerId;pad.setPointerCapture(pid);active=true;move(e);send(true);});
pad.addEventListener('pointermove',e=>{if(active&&e.pointerId===pid)move(e);});
function release(){active=false;x=0;y=0;setKnob();zeroSent=true;send(true);}
pad.addEventListener('pointerup',release);pad.addEventListener('pointercancel',release);
document.getElementById('stop').addEventListener('click',release);
document.addEventListener('visibilitychange',()=>{if(document.hidden)release();});
spd.addEventListener('input',()=>sv.textContent=spd.value);
function status(ok,ms){lastOk=ok?Date.now():lastOk;st.className=ok?'ok':'bad';
  st.textContent=ok?('已連線'+(ms!=null?' · 延遲 '+ms+' ms':'')):'連線中斷，車子會自動停止';}
async function send(force){
  if(busy&&!force)return;busy=true;const t0=performance.now();
  try{const r=await fetch('/cmd?x='+x.toFixed(2)+'&y='+y.toFixed(2)+'&s='+spd.value,{cache:'no-store'});
      status(r.ok,Math.round(performance.now()-t0));}
  catch(e){status(false);}
  busy=false;}
setInterval(()=>{if(active){zeroSent=false;send();}else if(!zeroSent){zeroSent=true;send(true);}},100);
setInterval(async()=>{if(active)return;try{const r=await fetch('/ping',{cache:'no-store'});status(r.ok);}catch(e){status(false);}},1000);
window.addEventListener('resize',setKnob);setKnob();
</script></body></html>)HTML";

void handleRoot() {
  server.send_P(200, "text/html; charset=utf-8", PAGE);
}

void handleCmd() {
  float x = constrain(server.arg("x").toFloat(), -1.0f, 1.0f);
  float y = constrain(server.arg("y").toFloat(), -1.0f, 1.0f);
  int s = server.arg("s").toInt();
  if (s > 0) maxPwm = constrain(s, MIN_PWM, 255);

  // 差速混合：y = 前後，x = 左右（往右推 → 左輪快、右輪慢）
  float l = y + x, r = y - x;
  float m = max(fabs(l), fabs(r));
  if (m > 1) { l /= m; r /= m; }
  int lp = toPwm(l), rp = toPwm(r);
  if (lp) lp += (lp > 0 ? BAL : -BAL);
  if (rp) rp -= (rp > 0 ? BAL : -BAL);

  drive(lp, rp);
  lastCmd = millis();
  moving = (lp != 0 || rp != 0);
  server.send(200, "text/plain", "ok");
}

void setup() {
  Serial.begin(115200);
  const int ins[] = {L_IN1, L_IN2, R_IN1, R_IN2};
  for (int p : ins) ledcAttach(p, PWM_FREQ, PWM_RES);
  drive(0, 0);

  WiFi.mode(WIFI_AP);
  WiFi.softAP(AP_SSID, AP_PASS);
  WiFi.setSleep(false);

  server.on("/", handleRoot);
  server.on("/cmd", handleCmd);
  server.on("/ping", []() { server.send(200, "text/plain", "ok"); });
  server.onNotFound([]() { server.sendHeader("Location", "/"); server.send(302, "text/plain", ""); });
  server.begin();

  Serial.println();
  Serial.printf("WiFi 熱點：%s  密碼：%s\n", AP_SSID, AP_PASS);
  Serial.print("手機連上後打開：http://");
  Serial.println(WiFi.softAPIP());
}

void loop() {
  server.handleClient();
  if (moving && millis() - lastCmd > CMD_TIMEOUT_MS) {
    drive(0, 0);
    moving = false;
    Serial.println("超過 0.5 秒沒收到指令，自動停車");
  }
}
```
{% endraw %}

</details>

## 心得

這是我第一次和朋友一起動手接線、焊接，從零開始組出一台機器人。過程中我對焊接和剝線都還不太熟練，花了不少時間摸索。

回頭檢討，這次最大的問題是**底盤規劃不夠周全**。底盤設計得太小，零件擠在一起，不只接線很不方便，也沒有多餘的空間放集塵盒和它的馬達。

另外，**線材也選錯了**。這次用的線太軟，反覆彎折時很容易斷掉，下次應該改用比較硬、比較耐折的線。

還有一點要說明：這次的**程式碼是 AI 協助寫的**，我目前還沒有仔細審過。我打算先讓車子的功能達到我想要的程度，再回頭好好審查、整理這份程式碼。

下一版我會先把底盤放大，預留集塵盒的位置，並換用更適合的線材，讓它更接近一台真正能掃地的機器人。
