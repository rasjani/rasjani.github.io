---
layout: default
title: index

---
{% for post in paginator.posts %}
  * [{{ post.title }}]({{ post.url | prepend: site.baseurl }})
    * {{ post.description }}
{% endfor %}
