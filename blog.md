---
layout: page
title: Blog
---

This is a collection of blog posts that I wrote about various topics. There isn't really any predominant theme or
reasoning behind them.

<ul>

{% for post in site.posts %}

<h2 class="post-title">
  {% unless post.category contains "rmarkdown" %}
  <a href="{{ post.url }}">{{ post.title }}</a>
</h2>

<span class="post-date">{{ post.date | date_to_string }}</span>

<details>
  <summary> 
    Show excerpt 
  </summary>

  {{ post.content | truncatewords:100 }}
  </p>
  <br>

</details>

<br>

  {% endunless %}

{% endfor %}
</ul>