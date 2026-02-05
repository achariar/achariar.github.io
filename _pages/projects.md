---
layout: page
title: projects
permalink: /projects/
description: a collection of cool projects 
nav: true
nav_order: 3
horizontal: false
---

<div class="projects">

{%- comment -%}
  Display ONLY projects in category "projects"
  Make sure each file in _projects has: category: projects
{%- endcomment -%}

{% assign sorted_projects = site.projects | where: "category", "projects" | sort: "importance" %}

{% if page.horizontal %}
  <div class="container">
    <div class="row row-cols-1 row-cols-md-2">
    {% for project in sorted_projects %}
      {% include projects_horizontal.liquid %}
    {% endfor %}
    </div>
  </div>
{% else %}
  <div class="row row-cols-1 row-cols-md-3">
    {% for project in sorted_projects %}
      {% include projects.liquid %}
    {% endfor %}
  </div>
{% endif %}

</div>
