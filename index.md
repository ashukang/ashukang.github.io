---
layout: page
title: 学无止境
tagline: Supporting tagline
---
{% include JB/setup %}

## 智能小设备

<ul class="posts">
  {% for post in site.categories."smart" %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

## 其他

<ul class="posts">
  {% for post in site.categories."other" %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>


