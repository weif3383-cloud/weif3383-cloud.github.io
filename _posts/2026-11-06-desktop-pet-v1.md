---
title: 桌面寵物・疊代一：透明視窗地基
categories: project
cover: /assets/images/projects/desktop-pet/cover.png
show_cover: false   # 封面只當卡片縮圖，文章內文不重複顯示
description: 用 Unity 做一隻會在桌面上活動的 AI 桌寵。疊代一只蓋最底層——一個背景全透明、永遠置頂、滑鼠能穿透的視窗。
---

這是一個用 Unity 做的 **AI 桌面寵物**：一隻之後會在桌面上走動、睡覺、被點會有反應的小角色，還能接 Claude API 跟它聊天（不想花錢就整個關掉，當純桌寵）。

疊代一（2026/11/5–11/6）只做一件事：**打地基**。因為整個專案技術風險最高的就是「透明視窗」——如果這關過不了，後面都不用談。所以先用一個**橘色暫代方塊**驗證技術，Live2D 模型最後才裝。

成果：桌面上一個橘色方塊，背景完全透明，只有方塊能點，其他地方的點擊會直接穿透到桌面。

![橘色暫代方塊浮在桌面上，背景完全透明，點擊會穿透到後面的桌布](/assets/images/projects/desktop-pet/transparent-window.jpg)
*暫代方塊浮在動態桌布上：方塊本體可以點、可以拖，其餘區域的點擊穿透到桌面*

## 規劃：先把方向定清楚

動手前先訂下幾個關鍵步驟：

| 主題 | 決定 | 為什麼 |
|---|---|---|
| 引擎 | Unity 6.4 + C#（2D） | 動畫狀態機強；Live2D 本身就是 2D，不需要 3D |
| 角色 | 遊走型（小章魚）／定點型（人形）兩種 | 人形在桌面到處走很怪，小吉祥物才適合亂跑 |
| AI | Claude API，每人自備 key，可全關 | 不用自己架後端；關掉就是純桌寵、不花錢 |
| 模型 | Live2D 最後才裝，先用方塊 | 先驗證風險最高的透明視窗 |
| 版權 | 官方範例模型 ＋ 從本機資料夾載入 | 手遊拆包模型、別家吉祥物不能公開散佈 |

## Unity 怎麼變成桌寵：四個步驟

一般的 Unity 遊戲是一個長方形視窗，要讓它變成「貼在桌面上的角色」，關鍵是四步：

1. **背景透明**：攝影機背景色的 alpha 設成 0，再請 Windows 的 DWM 把整個視窗當成「玻璃」——alpha 為 0 的地方就透出後面的桌面。
2. **蓋滿螢幕＋永遠置頂**：視窗做得跟整個螢幕一樣大、設成最上層，角色就能在整個桌面範圍活動。
3. **動態點擊穿透**：每一幀檢查滑鼠下面有沒有角色——有就接收點擊，沒有就讓這一幀的點擊穿過去打到桌面。這樣整片透明視窗才不會擋住你操作別的程式。
4. **角色本體**：這部分就跟一般 2D 遊戲一樣，用 Collider 判斷有沒有被點、用狀態機控制動作（留到疊代二）。

另外還要改幾個專案設定（關 DXGI Flip Model、關 URP HDR、繪圖 API 只留 Direct3D11、允許背景執行），少一個透明就會失敗。

## 三個最難的 bug：都只在打包後才現形

- **打包後整個螢幕全黑**：Unity 6 預設用 DirectX 12，而 DX12 的新式畫面輸出會把透明資訊丟掉。用排除法鎖定是繪圖 API 的問題，改成只留 Direct3D11 就好了。（脫困：Alt+F4 或工作管理員結束程式）
- **拖方塊時動態桌布會暫停**：點方塊會讓桌寵視窗取得「焦點」，桌布程式以為有全螢幕程式在用，就暫停省電。解法是加上「被點也不搶焦點」的視窗樣式（`WS_EX_NOACTIVATE`）。
- **桌面同時冒出兩個方塊**：兩個方塊能分開拖，代表是兩個 .exe 同時在跑。原本「只准一隻」的檢查只管同一程式內，改成直接向 Windows 登記一個全系統的具名鎖（`CreateMutexW`），第二隻發現鎖已存在就自己關掉。

抓到後兩個，靠的是**破壞性測試**：故意連開兩次、亂點滑鼠。

## 架構與重構

整個桌寵規劃成三層（平台層、純邏輯的 Core 層、畫面的 Presentation 層），疊代一只蓋了最底下的**平台層**。

寫到一半 Win32 的呼叫全擠在開場函式裡、「看得眼睛要瞎了」，就把它拆成兩個角色：一個決定「**什麼時候**」穿透，一個負責「**怎麼做**」這些 Windows 操作。判斷要不要拆的標準很簡單——名字能說出程式碼說不出的意圖，才值得拆。每個函式都補上說明註解。好讀比短更重要。

## Git 流程

分支分成 main（穩定）、dev（開發主線）、claude/…（AI 草稿）。一開始是 Claude 推到自己的分支、開 PR、我 merge 再 pull；後來改成 Claude 直接改我電腦的資料夾，我在 VS Code 的「原始檔控制」看差異，自己決定要不要 commit。`.gitignore` 特別把 API key 相關的檔案排除掉——金鑰一旦外洩只能作廢重辦。

<details markdown="1">
<summary>點我展開關鍵程式碼（C# / Win32）</summary>

{% raw %}
```csharp
// ① NativeWindow：透明、置頂、穿透
// 讓攝影機背景 alpha=0 的地方透出桌面
var wholeWindow = new Win32.MARGINS { Left = -1 };   // -1 = 整個視窗都延伸玻璃效果
Win32.DwmExtendFrameIntoClientArea(_hwnd, ref wholeWindow);

// 永遠置頂（位置和大小不變）
Win32.SetWindowPos(_hwnd, Win32.HWND_TOPMOST, 0,0,0,0,
  SWP_NOMOVE | SWP_NOSIZE | SWP_FRAMECHANGED);

// 每個旗標是一顆開關，用 | 合在一起
uint style = Win32.WS_EX_LAYERED;      // 可透明
style |= Win32.WS_EX_NOACTIVATE;       // 不搶焦點
if (clickThrough)
  style |= Win32.WS_EX_TRANSPARENT;    // 穿透
Win32.SetWindowLongPtr(_hwnd, GWL_EXSTYLE, (IntPtr)style);  // 64 位元要用 ...Ptr 版
```

```csharp
// ② WindowController：每一幀決定要不要穿透
private void Update() {
  ReadCursor();
  Vector3 world = _camera.ScreenToWorldPoint(CursorScreenPosition);
  IsOverInteractive = Physics2D.OverlapPoint(world) != null;
#if !UNITY_EDITOR && UNITY_STANDALONE_WIN
  bool want = !IsOverInteractive && !HoldInteractive;   // 滑鼠不在角色上就穿透
  if (want != _window.IsClickThrough)                   // 狀態有變才通知 Windows
    _window.SetClickThrough(want);
#endif
}
// 穿透時 Unity 收不到滑鼠，直接問 Windows；Y 軸要翻轉（Windows 原點在左上，Unity 在左下）
Win32.GetCursorPos(out var p);
Win32.ScreenToClient(_hwnd, ref p);
return new Vector2(p.X, Screen.height - p.Y);
```

```csharp
// ③ SingleInstance：只准一隻在跑
_handle = Win32.CreateMutexW(IntPtr.Zero, false,
  "Local\\DesktopPet_SingleInstance_7f3a");            // 向 Windows 登記一個具名鎖
bool alreadyExists = Marshal.GetLastWin32Error() == Win32.ERROR_ALREADY_EXISTS;
return _handle != IntPtr.Zero && !alreadyExists;       // 鎖已存在 → 第二隻自己關掉
```
{% endraw %}

</details>

## 下一步：疊代二

- **角色狀態機**：待機、走路、睡覺、點擊反應、被拖曳；再接上學習模式的第一個練習與測試。
- **右鍵選單＋存檔**：取代現在「右鍵＝關閉」的暫時設計（太容易誤觸），用 JSON 存下位置和名字。
- **換上 Live2D**：載入官方範例模型，從本機資料夾讀取。

完整的技術整理簡報放在這裡：[桌面寵物・疊代一 技術整理（PDF）](/assets/files/desktop-pet-iter1.pdf)

## 心得

第一次開發桌面寵物，有點不知道方向，所以想先把一個大目標拆成一個個零碎的小任務，再一一去實現。整個過程唯一比較麻煩的，也只有打包後那幾個需要小修的 bug。我相信自己一定還沒發現全部的 bug，這些就等之後給其他使用者使用時，再聽他們的回饋慢慢補。
