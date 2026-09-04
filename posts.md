---
layout: default
title: Blog
permalink: /posts/
---

# Blog

{% assign grouped_posts = site.posts | group_by_exp: "post", "post.date | date: '%Y'" %}

{% for year in grouped_posts %}

## {{ year.name }}

{% assign monthly = year.items | group_by_exp: "post", "post.date | date: '%B'" %}
{% for month in monthly %}

### {{ month.name }}

{% for post in month.items %}

- {{ post.date | date: "%b %d" }} – [{{ post.title }}]({{ post.url | relative_url }})
  {% endfor %}
  {% endfor %}
  {% endfor %}
