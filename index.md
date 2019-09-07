---
layout: archive
permalink: /
title: "Latest Posts"
image: 
    feature: /cover/tempcover.jpg
---

<div class="tile">
  <h2 class="post-title">更多</h2>
  <p class="post-excerpt">欢迎访问我的 <a href="https://github.com/HuskyTGame">Github</a>, 和<a href="http://huskytgame.lainedu.cn/">个人网站</a>, 更多信息待更新。</p>
</div><!-- /.tile -->
<div class="tile">
</div><!-- /.tile -->

<div class="tile">
</div><!-- /.tile -->

<div class="tile">
</div><!-- /.tile -->

<div class="tiles">
{% for post in site.posts %}
	{% include post-grid.html %}
{% endfor %}
</div><!-- /.tiles -->

