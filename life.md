---
layout: page
title: 生活紀錄
permalink: /life/
---

日常的小事、心得與隨筆。

{% if site.categories.life %}
{% for post in site.categories.life %}
- {{ post.date | date: "%Y-%m-%d" }}　[{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}
{% else %}
還沒有文章，敬請期待！
{% endif %}
