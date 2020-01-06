---
layout: page
title: projects
permalink: /projects/
description: stuff that we have been working on

---

{% for project in site.projects reversed %}
* [{{ project.title }}]({{ project.github }}) 
  * {{ project.description }}
  {% if project.documentation %}
  * [Documentation]({{ project.documentation}}) 
  {% endif %}
{% endfor %}
