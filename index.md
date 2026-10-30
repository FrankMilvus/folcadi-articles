---
layout: page
title: Folcadi articles
---

Write-ups about building **Folcadi** — a language-exchange messenger with chat,
voice and video calls, media messages, and on-the-fly translation and
transcription. Flutter on the client, FastAPI + PostgreSQL on the server.

## Posts

{% assign posts = site.pages | where_exp: "p", "p.path contains 'articles/'" | sort: "path" | reverse %}
<ul class="post-list" style="list-style:none;padding:0;">
{% for post in posts %}
  <li style="margin-bottom:1.75rem;">
    {%- if post.image %}
    <a href="{{ post.url | relative_url }}">
      <img src="{{ post.image }}" alt="{{ post.title | escape }}"
           style="width:100%;max-width:420px;height:auto;border-radius:6px;display:block;margin-bottom:.5rem;">
    </a>
    {%- endif %}
    <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
    {%- if post.description %}
    <p style="margin:.25rem 0 0;">{{ post.description }}</p>
    {%- endif %}
  </li>
{% endfor %}
</ul>
