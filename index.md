---
layout: default
title: Summer Sun
---
<div>
<hgroup>
    <h1 class="site-title">
      <a href="/" title="Everything">Radom thoughts</a></h1>
    <h1 class="site-description">coding, reading, travelling...</h1>
  </hgroup>
  <div class="entry">
    {% for post in site.posts %}
    <div class="date">{{ post.date | date: "%Y-%m-%d" }}</div>
    <h1><a href="{{ post.url }}">{{ post.title }}</a></h1>
    {% endfor %}
    </div>
    <footer class="entry-meta">
    </footer>
</div>