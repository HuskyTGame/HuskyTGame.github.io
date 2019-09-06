---
layout: archive
permalink: /basics/
title: "基础"
excerpt: "Dedicate all my life to my love"
---

<div class="tiles">
{% for post in site.posts %}
	{% if post.categories contains 'basics' %}
		{% include post-grid.html %}
	{% endif %}
{% endfor %}
</div><!-- /.tiles -->


