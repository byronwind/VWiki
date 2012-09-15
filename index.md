---
layout: index 
---

{% for post in site.posts %}

## [{{ post.title }}]({{ post.url }}) 
<time>{{ post.date | date_to_long_string }}</time> By:{{ site.author_info }}

Tags: {{ post.tags | join: ', ' }}

{{post.summary}}

[Read More &raquo;]({{ post.url }})

----

{% endfor %}