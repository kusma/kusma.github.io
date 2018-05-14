---
layout: page
title: Blog
permalink: /blog/
---

Welcome to my blog! I'll be writing

{% if site.posts.size > 0 %}
<h2>Recent blog posts:</h2>
{% for post in site.posts %}
<div class="card mb-3">
  <div class="card-body">
    <h3 class="card-title">{{ post.title | escape }}</h3>
    <span class="card-subtitle mb-2 text-muted">{{ post.date | date: "%b %-d, %Y" }}</span>

    <p class="card-text">{{ post.excerpt }}</p>
    <a class="card-link" href="{{ post.url }}">Read more</a>
  </div>
</div>
{% endfor %}
{% else %}
There's sadly no blogs here yet :(
{% endif %}
