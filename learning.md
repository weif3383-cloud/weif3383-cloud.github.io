---
layout: page
title: 學習紀錄
permalink: /learning/
---

正在學的東西都整理在這裡，依主題分區。

{% comment %}
  分區清單在 _data/learning.yml，每個分區一張卡片，點進去是該分區的文章列表。
{% endcomment %}
<div class="project-grid learning-grid">
{%- for section in site.data.learning -%}
  {%- assign notes = site.categories[section.id] -%}
  <a class="project-card" href="{{ "/learning/" | append: section.id | append: "/" | relative_url }}">
    <div class="project-thumb">
      {%- if section.cover -%}
      <img src="{{ section.cover | relative_url }}" alt="{{ section.name }}" loading="lazy">
      {%- else -%}
      <span class="project-thumb-text">{{ section.icon | default: section.name | slice: 0 }}</span>
      {%- endif -%}
    </div>
    <div class="project-info">
      <h3 class="project-title">{{ section.name }}</h3>
      <p class="project-desc">{{ section.description }}</p>
      <span class="post-meta">{% if notes %}{{ notes.size }} 篇{% else %}還沒有文章{% endif %}</span>
    </div>
  </a>
{%- endfor -%}
</div>
