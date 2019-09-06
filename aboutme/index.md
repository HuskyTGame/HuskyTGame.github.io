---
layout: archive
permalink: /aboutme/
title: "About Me"
excerpt: "Dedicate all my life to my love"
---

<div class="tiles">
{% for post in site.posts %}
	{% if post.categories contains 'aboutme' %}
		{% include post-grid.html %}
	{% endif %}
{% endfor %}
</div><!-- /.tiles -->



