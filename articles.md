---
layout: default
title: Articles
---

# Articles

Put some articles here.

{% for article in site.articles %}

## [{{ article.name }}]({{article.url}})

[Look at](https://jekyllrb.com/docs/pages/)

{% endfor %}