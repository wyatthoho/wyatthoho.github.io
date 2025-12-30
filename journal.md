---
layout: default
title: Journal
---

# Journal

{% for entry in site.journal %}

## [{{ entry.name }}]({{ entry.url }})

{{ entry.excerpt }}

{% endfor %}