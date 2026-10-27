---
layout: page
title: 專案紀錄
permalink: /projects/
---

做過的東西都整理在這裡，最新的在最上面。

{% comment %}
  每個專案是一張卡片：上面是封面圖（文章的 cover），下面是標題、簡介（description）和日期。
  點卡片會進到專案的詳細介紹。沒有設定封面時，會用標題第一個字當封面。
{% endcomment %}
{%- assign projects = site.categories.project -%}
{%- if projects and projects.size > 0 -%}
<div class="project-grid">
{%- for post in projects -%}
  <a class="project-card" href="{{ post.url | relative_url }}">
    <div class="project-thumb">
      {%- if post.cover -%}
      <img src="{{ post.cover | relative_url }}" alt="{{ post.title | escape }}" loading="lazy">
      {%- else -%}
      <span class="project-thumb-text">{{ post.title | slice: 0 }}</span>
      {%- endif -%}
    </div>
    <div class="project-info">
      <h3 class="project-title">{{ post.title | escape }}</h3>
      <p class="project-desc">{{ post.description | default: post.excerpt | strip_html | strip_newlines | truncate: 60 }}</p>
      <span class="post-meta">{{ post.date | date: "%Y-%m-%d" }}</span>
    </div>
  </a>
{%- endfor -%}
</div>
{%- else -%}
<p class="post-empty">還沒有文章，敬請期待！</p>
{%- endif -%}
