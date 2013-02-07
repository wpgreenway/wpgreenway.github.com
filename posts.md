---
layout: page
---

<h1>Posts</h1>

<p>I'm drawing a distinction here between tutorials and "other writing". As a personal development goal, I want to make sure that I'm writing regularly. A byproduct of that is that the signal:noise will likely be pretty high. I will be putting more time and effort into tutorials and want to keep them more readily available.</p>

<h2>Tutorials</h2>
<div class="posts">
{% for post in site.categories.tutorials %}
    {% include site/post_link.html %}
{% endfor %}
</div>

<h2>Writing</h2>
<div class="posts">
{% for post in site.categories.writing %}
    {% include site/post_link.html %}
{% endfor %}
</div>