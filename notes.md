---
layout: default
title: Notes
---

# Notes

{% for note in site.notes %}

## [{{ note.name }}]({{ note.url }})
Published {{ note.birth | date_to_string: "ordinal", "US" }} · Updated {{ note.date | date_to_string: "ordinal", "US" }}

---

{% endfor %}