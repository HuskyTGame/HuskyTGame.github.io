---
layout: archive
permalink: /shader/
title: "着色器"
excerpt: "Dedicate all my life to my love"
---

<div class="tiles">
{% for post in site.posts %}
	{% if post.categories contains 'shader' %}
		{% include post-grid.html %}
	{% endif %}
{% endfor %}
</div><!-- /.tiles -->


