---
layout: default
title: "Tags"
permalink: /tags/
---
<div class="page-content">
  <h1 class="page-title">tags</h1>
  {% assign all_tags = site.posts | map: 'tags' | join: ',' | split: ',' | uniq | sort %}
  <div class="tags-cloud">
    {% for tag in all_tags %}{% if tag != "" %}<a href="#{{ tag }}" class="tag-item">{{ tag }}</a>{% endif %}{% endfor %}
  </div>
  {% for tag in all_tags %}{% if tag != "" %}
  <div id="{{ tag }}" style="margin-top:2.2rem;padding-top:1rem;border-top:1px solid #281d4a">
    <p style="font-size:.65rem;color:#6b54a0;letter-spacing:2px;text-transform:uppercase;margin-bottom:.8rem">#{{ tag }}</p>
    <ul class="post-list">
      {% for post in site.posts %}{% if post.tags contains tag %}
      <li class="post-item">
        <div class="post-title"><a href="{{ post.url | relative_url }}">{{ post.title }}</a></div>
        <div class="post-date">{{ post.date | date: "%d.%m.%Y" }}</div>
      </li>
      {% endif %}{% endfor %}
    </ul>
  </div>
  {% endif %}{% endfor %}
</div>
