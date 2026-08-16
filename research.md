---
layout: default
title: Research
permalink: /research/
description: "Publications and working papers by Sakshi Bhalla."
---

## Publications

{% assign published = site.data.publications | where_exp: "item", "item.status != 'wip' and item.status != 'forthcoming'" | sort: "year" | reverse %}
{% assign forthcoming = site.data.publications | where: "status", "forthcoming" %}
{% for item in forthcoming %}{% include publication.html pub=item %}{% endfor %}
{% for item in published %}{% include publication.html pub=item %}{% endfor %}

## Works in progress

{% for item in site.data.publications %}{% if item.status == 'wip' %}{% include publication.html pub=item %}{% endif %}{% endfor %}
