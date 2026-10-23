---
layout: page
title: 關於我
permalink: /about/
---

目前就讀台北科技大學資工系一年級。

興趣是畫畫、打遊戲和旅遊。打的遊戲有原神、崩鐵、絕區零和 Minecraft，如果有人想要一起玩，歡迎來找我！

## 興趣與技能

我會一點鋼琴和直笛，直笛學得十分有心得，如果有興趣，我也可以手把手教你！

目前正在鑽研程式語言、研究機器人怎麼開發，也在學習資訊安全。如果有人在這方面有研究，歡迎認識一起研究 :D 之後可能會輔系電子系。

## 聯絡我

{% comment %}
  按鈕連結：
  - Discord：點了會複製使用者名稱（data-copy），對方貼到 Discord 的「新增好友」就能加你
  - Instagram：https://www.instagram.com/帳號/
{% endcomment %}
<div class="contact-links">
  <a class="contact-btn" href="https://github.com/weif3383-cloud" target="_blank" rel="noopener">GitHub</a>
  <button type="button" class="contact-btn" data-copy="thenoobsteve">Discord：thenoobsteve</button>
  <a class="contact-btn" href="https://www.instagram.com/thenoobjelly/" target="_blank" rel="noopener">Instagram</a>
</div>

<p class="image-credit">首頁輪播圖出處：流螢 — <a href="https://www.pixiv.net/artworks/136379199" target="_blank" rel="noopener">Pixiv #136379199</a>。其他圖片收集自網路，如果你是作者並希望標註或移除，歡迎聯絡我。</p>

<script>
  // 點 Discord 按鈕時複製使用者名稱，按鈕文字會暫時變成「已複製！」
  document.querySelectorAll(".contact-btn[data-copy]").forEach(function (btn) {
    var label = btn.textContent;
    btn.addEventListener("click", function () {
      navigator.clipboard.writeText(btn.dataset.copy).then(function () {
        btn.textContent = "已複製！到 Discord 新增好友貼上";
        setTimeout(function () { btn.textContent = label; }, 2500);
      });
    });
  });
</script>
