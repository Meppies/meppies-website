---
layout: page
title: Writing
description: All posts on meppie.eu — EUC, Infrastructure as Code, DevOps, automation and the occasional side project.
permalink: /articles/
---

Every post, newest first. Grouped by year.

{% assign posts_by_year = site.posts | group_by_exp: "post", "post.date | date: '%Y'" %}

<div class="archive-list">
{% for year in posts_by_year %}
  <h2 class="display display-sm" style="margin-top: 48px; border-bottom: 1px solid var(--border); padding-bottom: 8px;">{{ year.name }}</h2>
  {% for post in year.items %}
    <a href="{{ post.url | relative_url }}" class="archive-item">
      <h3>{{ post.title }}</h3>
      <span class="date">{{ post.date | date: '%d %b' }}</span>
    </a>
  {% endfor %}
{% endfor %}
</div>
