---
layout: archive
title: "Research & Experience"
permalink: /research/
author_profile: true
---

Click headings to explore all projects within the Organisations!


{% include base_path %}

{% assign research_items = site.research | sort: "order" %}
{% for post in research_items %}
  {% include archive-single.html %}
{% endfor %}
