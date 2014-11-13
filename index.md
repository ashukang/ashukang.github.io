---
layout: page
title: 学无止境
tagline: Supporting tagline
---
{% include JB/setup %}

## 其他

<ul class="posts">
  {% for post in site.categories."other" %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>


