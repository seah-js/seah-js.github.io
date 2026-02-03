---
layout: page
title: Agents
permalink: /agents/
---

How AI agents work — tool use, reasoning, planning, and building them.

{% assign agent_posts = site.posts | where_exp: "post", "post.categories contains 'agents'" %}
{% for post in agent_posts %}
- [{{ post.title }}]({{ post.url }}) — {{ post.description }}
{% endfor %}

{% if agent_posts.size == 0 %}
*First post coming soon.*
{% endif %}
