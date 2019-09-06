---
layout: archive
permalink: /
title: "Latest Posts"
image: 
    feature: /cover/tempcover.jpg
---

<div class="tiles">
{% for post in site.posts %}
	{% include post-grid.html %}
{% endfor %}
</div><!-- /.tiles -->
