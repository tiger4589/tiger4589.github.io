---
layout: default
title: Home
---

## Recent posts

{% for post in site.posts limit:5 %}

## [{{ post.title }}]({{ post.url | relative_url }})

_{{ post.date | date: "%d %B %Y" }}_

{{ post.excerpt }}

[Read more →]({{ post.url | relative_url }})

---

{% endfor %}

---

➡️ [Browse all posts]({{ '/posts/' | relative_url }})
