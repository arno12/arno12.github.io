---
layout: page
title: Data
---

As a data analyst, I often find myself trying out new things, be it about modeling or visualization. 
In this section, I will share the things I find out, either useful or just interesting.
The collection of posts below were written in [R Markdown](https://rmarkdown.rstudio.com/)

{% for post in site.posts %}
    {% if post.category contains "rmarkdown" %}

<h2 class="post-title">
<a href="{{ post.url }}">{{ post.title }}</a>
</h2>

<span class="post-date">
{{ post.date | date_to_string }}
</span>

    {% endif %}
{% endfor %}
