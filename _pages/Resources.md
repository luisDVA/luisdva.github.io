---
title: "Resources"
permalink: /resources/
layout: category
taxonomy: practices
entries_layout: list
author_profile: true
---
Select a practice to view its content.

{% assign practice_posts = site.posts | where_exp:"post", "post.categories contains 'practice'" %}
<ul>
  {% for post in practice_posts %}
    <li>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a> - {{ post.date | date: "%Y-%m-%d" }}
    </li>
  {% endfor %}
</ul>
