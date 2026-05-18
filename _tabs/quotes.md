---
layout: page
icon: fas fa-quote-left
order: 5
---

这里记录每天遇到的、有启发的一句话。

{% assign quote_posts = site.posts | where_exp: "post", "post.categories contains 'Quotes'" %}

{% if quote_posts.size > 0 %}
## 文章列表

{% for post in quote_posts %}
- [{{ post.title }}]({{ post.url }}) — {{ post.date | date: "%Y-%m-%d" }}
{% endfor %}
{% else %}

还没有内容，先从今天开始。

{% endif %}
