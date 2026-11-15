---
layout: page
title: 專案紀錄
permalink: /projects/
---

做過的東西都整理在這裡，最新的在最上面。

{% comment %}
  兩種卡片：
  1. 多版本專案（_data/projects.yml）：一張卡片進去看各版本，像掃地機器人。
  2. 單篇專案（categories: project）：點卡片直接進到那篇介紹。
  沒有封面時，用名稱第一個字當封面。
{% endcomment %}
<div class="project-grid">
{%- for group in site.data.projects -%}
  {%- assign posts = site.categories[group.id] -%}
  <a class="project-card" href="{{ "/projects/" | append: group.id | append: "/" | relative_url }}">
    <div class="project-thumb">
      {%- if group.cover -%}
      <img src="{{ group.cover | relative_url }}" alt="{{ group.name }}" loading="lazy">
      {%- else -%}
      <span class="project-thumb-text">{{ group.name | slice: 0 }}</span>
      {%- endif -%}
    </div>
    <div class="project-info">
      <h3 class="project-title">{{ group.name }}</h3>
      <p class="project-desc">{{ group.description }}</p>
      <span class="post-meta">{% if posts %}{{ posts.size }} 篇{% else %}還沒有文章{% endif %}</span>
    </div>
  </a>
{%- endfor -%}
{%- assign projects = site.categories.project -%}
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
