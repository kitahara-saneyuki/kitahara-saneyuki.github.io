---
title: categories
---

All blog posts, grouped by categories.

{% assign categories = site.categories | sort %}

{% for category in categories %}
<h3>#{{ category[0] }}</h3>

{% for post in category[1] %}
- [{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}
{% endfor %}
