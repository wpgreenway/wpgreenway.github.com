---
layout: page
---

<h2>Tutorials</h2>
<div class="posts">
{% for post in site.categories.tutorials %}
    {% include site/post_link.html %}
{% endfor %}
</div>

<h2>Other</h2>
<div class="posts">
{% for post in site.categories.writing %}
    {% include site/post_link.html %}
{% endfor %}
</div>