---
layout: default
title: Blog
permalink: /posts/
---

# Blog

{% assign grouped_posts = site.posts | group_by_exp: "post", "post.date | date: '%Y'" %}

{% for year in grouped_posts %}

## {{ year.name }}

{% for post in year.items %}

- {{ post.date | date: "%b %d" }} –
  [{{ post.title }}]({{ post.url | relative_url }})
  {% endfor %}

{% endfor %}
