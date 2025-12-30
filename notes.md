---
layout: default
title: Notes
---

# Notes

{% for note in site.notes %}

## [{{ note.name }}]({{ note.url }})

{{ note.excerpt }}
{% endfor %}