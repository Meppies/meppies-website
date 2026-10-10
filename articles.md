---
layout: page
title: Writing
description: All posts on meppie.eu — EUC, Infrastructure as Code, DevOps, automation and the occasional side project.
permalink: /articles/
label: 01 / Writing
lede: Every post, newest first, grouped by year.
---


{% assign posts_by_year = site.posts | group_by_exp: "post", "post.date | date: '%Y'" %}

<div class="archive-list">
{% for year in posts_by_year %}
  <h2 class="archive-year">{{ year.name }}</h2>
  {% for post in year.items %}
    <a href="{{ post.url | relative_url }}" class="archive-item">
      <span>{{ post.title }}</span>
      <span class="date">{{ post.date | date: '%-d %b' }}</span>
    </a>
  {% endfor %}
{% endfor %}
</div>
