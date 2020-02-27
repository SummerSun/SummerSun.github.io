---
layout: default
title: Summer Sun
---
<div class="entry">

  {% for post in site.posts %}
    <div class="date">{{ post.date | date: "%Y-%m-%d" }}</div>
      <h1><a href="{{ post.url }}">{{ post.title }}</a></h1>
  {% endfor %}
</div>