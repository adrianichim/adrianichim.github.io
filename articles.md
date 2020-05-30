---
layout: page
title: Technical Articles
subtitle: List of my technical articles
---

![Hic Sunt Leones]({{site.baseurl}}/assets/img/hic-sunt-leones-02.jpg)  

###### Hic sunt leones

{% for article in site.articles %}
> {{article.date | date_to_string}} [{{article.title}}]({{article.url}})
{% endfor %}




