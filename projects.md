---
layout: page
title: Projects
permalink: /projects/
description: stuff that we have been working on

---

<ul class="project-list">
{% for project in site.projects reversed %}
  <div class="project-container">
    <div class="project-name">
        <a class="project-title" href="{{ project.github }}">{{ project.title }}</a>
    </div>
    <div class="project-description">
        {{ project.description }}
    </div>
    {% if project.documentation %}
    <div class="project-documentation">
        <a class="project-documentation-link" href="{{ project.documentation}}">[Documentation]</a>
    </div>
    {% endif %}
  </div>
{% endfor %}
