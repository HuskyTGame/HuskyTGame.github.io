---
layout: archive
permalink: /ai/
title: "Game AI"
excerpt: "Dedicate all my life to my love"
---

<div class="tiles">
{% for post in site.posts %}
	{% if post.categories contains 'ai' %}
		{% include post-grid.html %}
	{% endif %}
{% endfor %}
</div><!-- /.tiles -->


