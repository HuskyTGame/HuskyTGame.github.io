---
layout: archive
permalink: /others/
title: "杂记"
excerpt: "Dedicate all my life to my love"
---

<div class="tiles">
{% for post in site.posts %}
	{% if post.categories contains 'others' %}
		{% include post-grid.html %}
	{% endif %}
{% endfor %}
</div><!-- /.tiles -->


