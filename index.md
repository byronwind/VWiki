---
layout: index 
---

{% for post in site.posts %}

## [{{ post.title }}]({{site.base_url}}{{ post.url }}) 
<time>{{ post.date | date_to_long_string }}</time> By:{{ site.author_info }}

Tags: {{ post.tags | join: ', ' }}

{{post.summary}}

[Read More &raquo;]({{site.base_url}}{{ post.url }})

----

{% endfor %}