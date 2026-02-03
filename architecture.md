---
layout: page
title: Architecture
permalink: /architecture/
---

Deep dives into how models actually work — from attention mechanisms to mixture of experts.

{% assign arch_posts = site.posts | where_exp: "post", "post.categories contains 'architecture'" %}
{% for post in arch_posts %}
- [{{ post.title }}]({{ post.url }}) — {{ post.description }}
{% endfor %}

{% if arch_posts.size == 0 %}
*First post coming soon.*
{% endif %}
