---
layout: page
title: Infrastructure
permalink: /infra/
---

The engineering side — serving models, scaling, MLOps, and making things fast.

{% assign infra_posts = site.posts | where_exp: "post", "post.categories contains 'infra'" %}
{% for post in infra_posts %}
- [{{ post.title }}]({{ post.url }}) — {{ post.description }}
{% endfor %}

{% if infra_posts.size == 0 %}
*First post coming soon.*
{% endif %}
