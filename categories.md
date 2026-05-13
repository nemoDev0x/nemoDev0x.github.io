---
layout: default
title: "Categorías"
permalink: /categories/
---
<div class="page-content">

  <h1 class="page-title">categorías</h1>

  {% assign cat_list = site.posts | map: 'categories' | join: ',' | split: ',' | uniq | sort %}

  <!-- Resumen de categorías -->
  <div class="cat-grid">
    {% for cat in cat_list %}
    {% if cat != "" %}
    {% assign cat_posts = site.posts | where_exp: "post", "post.categories contains cat" %}
    <a href="#{{ cat | slugify }}" class="cat-card">
      <div class="cat-card-icon">
        {% case cat %}
          {% when 'reconocimiento' %}⬡
          {% when 'ciberseguridad' %}⬡
          {% when 'redes' %}⬡
          {% when 'web' %}⬡
          {% when 'active-directory' %}⬡
          {% when 'post-explotacion' %}⬡
          {% when 'forense' %}⬡
          {% when 'osint' %}⬡
          {% when 'herramientas' %}⬡
          {% when 'ctf' %}⬡
          {% else %}⬡
        {% endcase %}
      </div>
      <div class="cat-card-name">{{ cat }}</div>
      <div class="cat-card-count">{{ cat_posts.size }} posts</div>
    </a>
    {% endif %}
    {% endfor %}
  </div>

  <!-- Posts agrupados por categoría -->
  {% for cat in cat_list %}
  {% if cat != "" %}
  {% assign cat_posts = site.posts | where_exp: "post", "post.categories contains cat" %}
  {% if cat_posts.size > 0 %}

  <div id="{{ cat | slugify }}" class="cat-section">
    <div class="cat-section-header">
      <span class="cat-section-title">{{ cat }}</span>
      <span class="cat-section-badge">{{ cat_posts.size }}</span>
    </div>

    <ul class="post-list">
      {% for post in cat_posts %}
      <li class="post-item">
        <div>
          <div class="post-title"><a href="{{ post.url | relative_url }}">{{ post.title }}</a></div>
          {% if post.tags %}
          <div class="post-tags">
            {% for tag in post.tags limit:5 %}
            <a href="{{ '/tags/#' | append: tag | relative_url }}" class="tag">{{ tag }}</a>
            {% endfor %}
          </div>
          {% endif %}
        </div>
        <div class="post-date">{{ post.date | date: "%d.%m.%Y" }}</div>
      </li>
      {% endfor %}
    </ul>
  </div>

  {% endif %}
  {% endif %}
  {% endfor %}

</div>
