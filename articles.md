---
layout: page
title: Articles
permalink: /articles/
description: Articles Matheus has published on other sites and publications.
---

Pieces I've published elsewhere. The blog's own posts are on the
[home page](/).

The list below comes from `_data/articles.yml`.

{% for article in site.data.articles %}
### [{{ article.title }}]({{ article.url }})

*{{ article.publication }} · {{ article.date | date: "%B %-d, %Y" }}*

{{ article.summary }}
{% endfor %}
