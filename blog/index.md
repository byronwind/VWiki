---
layout: page
title: Aticles
---


{{ page.title }}
================

{% for post in site.categories.blog %}

## [{{ post.title }}]({{site.base_url}}{{ post.url }}) 
<time>{{ post.date | date_to_long_string }}</time> By:{{ site.author_info }}

Tags: {{ post.tags | join: ', ' }}

{{post.summary}}

[Read More &raquo;]({{site.base_url}}{{ post.url }}) 

----

{% endfor %}