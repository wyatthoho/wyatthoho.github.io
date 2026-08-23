---
layout: default
title: CAE Hacks
---

# CAE Hacks

{% assign sorted_notes = site.cae_hacks | sort: "birth" | reverse %}

{% for note in sorted_notes %}

## [{{ note.name }}]({{ note.url }})
Published {{ note.birth | date_to_string: "ordinal", "US" }} 
· Updated {{ note.date | date_to_string: "ordinal", "US" }} 
· {{ site.author }}

---

{% endfor %}