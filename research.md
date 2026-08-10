---
layout: default
title: Research
permalink: /research/
description: "Publications and working papers by Sakshi Bhalla."
---

My current and forthcoming work focuses on structural change in information environments, media and information infrastructures and their consequences for political behavior, and platform-mediated news exposure and polarization.

## Publications

{% assign pubs = site.data.publications | where_exp: "item", "item.status != 'wip'" | sort: "year" | reverse %}
{% for item in pubs %}{% include publication.html pub=item %}{% endfor %}

## Works in progress

{% for item in site.data.publications %}{% if item.status == 'wip' %}{% include publication.html pub=item %}{% endif %}{% endfor %}
