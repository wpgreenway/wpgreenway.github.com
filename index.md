---
layout: page
---

<h1>Posts</h1>

{% for post in site.posts %}
<ul class="posts">
  <a href="{{post.url}}">{{ post.title }}</a> - {{ post.date | date: "%B %e, %Y" }}
</ul>
{% endfor %}