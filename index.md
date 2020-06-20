---
layout: page
title: YANG's blog
tagline: Yet ANother Good blog
---

这个人很懒，只写了 `{{ site.posts | size }}` 篇文章。

## Latest Posts
<ul class="post-list">
  {% for post in site.posts %}
    <li>
      {% assign date_format = site.cayman-blog.date_format | default: "%b %-d, %Y" %}
      <span class="post-meta">{{ post.date | date: date_format }}</span>
      <h2>
        <a class="post-link" href="{{ post.url | absolute_url }}" title="{{ post.title }}">{{ post.title | escape }}</a>
      </h2>    
    </li>
  {% endfor %}
</ul>