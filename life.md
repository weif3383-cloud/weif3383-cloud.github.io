---
layout: page
title: 生活紀錄
permalink: /life/
---

日常的小事、心得與隨筆。

{% for post in site.categories.life %}
- {{ post.date | date: "%Y-%m-%d" }}　[{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}
