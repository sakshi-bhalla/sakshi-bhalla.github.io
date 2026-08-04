---
layout: default
title: Research
permalink: /research/
description: "Publications and working papers by Sakshi Bhalla."
---

My current and forthcoming work focuses on structural change in information environments, media and information infrastructures and their consequences for political behavior, and platform-mediated news exposure and polarization.

## Publications

{% for item in site.data.publications %}{% unless item.status == 'wip' %}{% include publication.html pub=item %}{% endunless %}{% endfor %}

## Works in progress

{% for item in site.data.publications %}{% if item.status == 'wip' %}{% include publication.html pub=item %}{% endif %}{% endfor %}
