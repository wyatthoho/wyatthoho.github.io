---
layout: default
title: Dev Notes
---

# Dev Notes

{% assign sorted_notes = site.dev_notes | sort: "birth" | reverse %}

{% for note in sorted_notes %}

## [{{ note.name }}]({{ note.url }})
Published {{ note.birth | date_to_string: "ordinal", "US" }} 
· Updated {{ note.date | date_to_string: "ordinal", "US" }} 
· {{ site.author }}

---

{% endfor %}