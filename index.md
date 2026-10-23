---
layout: default
---

{% comment %}
  首頁輪播圖：把圖片或動圖丟進 assets/images/hero/ 資料夾就會自動輪播，
  依檔名排序，例如 01.png、02.gif、03.mp4。建議尺寸 1920x1080（16:9）。
  支援：png、jpg、webp、gif，以及 mp4、webm 影片（會自動靜音循環播放，像動圖一樣）。
  資料夾是空的時候會顯示提示文字。
{% endcomment %}
{% assign hero_images = "" | split: "" %}
{% for f in site.static_files %}{% if f.path contains '/assets/images/hero/' %}{% assign ext = f.extname | downcase %}{% if ext == ".png" or ext == ".jpg" or ext == ".jpeg" or ext == ".webp" or ext == ".gif" or ext == ".mp4" or ext == ".webm" %}{% assign hero_images = hero_images | push: f %}{% endif %}{% endif %}{% endfor %}
{% assign hero_images = hero_images | sort: "path" %}
<div class="home">
{% if hero_images.size > 0 %}
<div class="hero-slider" data-interval="4000">
{% for img in hero_images %}{% assign ext = img.extname | downcase %}
{% if ext == ".mp4" or ext == ".webm" %}
  <video class="hero-slide{% if forloop.first %} is-active{% endif %}" src="{{ img.path | relative_url }}" muted loop playsinline preload="{% if forloop.first %}auto{% else %}metadata{% endif %}"{% if forloop.first %} autoplay{% endif %}></video>
{% else %}
  <img class="hero-slide{% if forloop.first %} is-active{% endif %}" src="{{ img.path | relative_url }}" alt="{{ img.basename }}" decoding="async">
{% endif %}
{% endfor %}
{% if hero_images.size > 1 %}
  <div class="hero-dots">
  {% for img in hero_images %}<button type="button" aria-label="第 {{ forloop.index }} 張"{% if forloop.first %} class="is-active"{% endif %}></button>{% endfor %}
  </div>
{% endif %}
</div>
{% else %}
<div class="hero-slider hero-placeholder">放圖片（建議 1920×1080）</div>
{% endif %}

{% comment %} 最新更新：只顯示最新 4 篇（改 limit 後面的數字就能調整），沒有文章時顯示「無」 {% endcomment %}
<h2 class="post-list-heading">最新更新</h2>
{% if site.posts.size == 0 %}
<p class="post-empty">無</p>
{% else %}
<ul class="post-list">
{% for post in site.posts limit: 4 %}
  <li>
    <span class="post-meta">{{ post.date | date: "%Y-%m-%d" }}</span>
    <h3><a class="post-link" href="{{ post.url | relative_url }}">{{ post.title | escape }}</a></h3>
  </li>
{% endfor %}
</ul>
{% endif %}
</div>

<script>
  // 首頁輪播：每隔幾秒換下一張，點小圓點可以直接跳到那一張
  (function () {
    var slider = document.querySelector(".hero-slider[data-interval]");
    if (!slider) return;
    var slides = slider.querySelectorAll(".hero-slide");
    var dots = slider.querySelectorAll(".hero-dots button");
    if (slides.length < 2) return;
    var current = 0;
    var timer = null;
    var interval = parseInt(slider.dataset.interval, 10) || 4000;

    function show(i) {
      slides[current].classList.remove("is-active");
      dots[current].classList.remove("is-active");
      if (slides[current].tagName === "VIDEO") slides[current].pause();
      current = (i + slides.length) % slides.length;
      slides[current].classList.add("is-active");
      dots[current].classList.add("is-active");
      // 影片只在輪到它的時候播放，省流量也比較順
      if (slides[current].tagName === "VIDEO") slides[current].play().catch(function () {});
    }
    function start() {
      if (window.matchMedia("(prefers-reduced-motion: reduce)").matches) return;
      stop();
      timer = setInterval(function () { show(current + 1); }, interval);
    }
    function stop() { clearInterval(timer); }

    dots.forEach(function (dot, i) {
      dot.addEventListener("click", function () { show(i); start(); });
    });
    slider.addEventListener("mouseenter", stop);
    slider.addEventListener("mouseleave", start);
    start();
  })();
</script>
