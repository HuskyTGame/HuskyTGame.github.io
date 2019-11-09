---
layout: article
title:  "我的独立博客"
categories: others
image:
    teaser: /in-post/others/2019-09-06-StartHere/StartHere.jpg
---

## 写在前面

一直有写笔记的习惯，喜欢将学习过的知识总结记录下来。其实一开始主观上我并不想这么做，但为了降低复习成本，我慢慢喜欢上了写笔记。再往后，写笔记成为了我保障学习质量的一个方法。现在，为了继续提高学习质量和学习动力，打算开始写自己的博客，记录自己学习过的知识以及自己的一些思考。由于平日经常混迹博客园、CSDN、知乎、简书等地，对各类博客也有一些了解。个人来说非常喜欢简约一点的博客风格，能最大化凸显文章内容是我最为优先考虑的。思考再三决定在GitHub上搭建一个静态博客。

## 搭建博客

搭建此博客主要参考了以下链接：
* https://643435675.github.io/2015/02/15/create-my-blog-with-jekyll/
* https://blog.csdn.net/xudailong_blog/article/details/78762262
* https://candycat1992.github.io/2016/03/02/hello-blog/

## 写在最后

以后会在此博客中更新着色器、游戏框架、编辑器小工具、小游戏制作等内容。

## 维护博客中注意事项

- 1.新增*categories*分类方式：（以新增Game分类为例）

  Step1：根目录新增文件夹，文件夹命名为*url*名称（eg：game）

  Step2：根目录game文件夹下添加*index.md*文件

  ````
  ---
  layout: archive
  permalink: /game/
  title: "游戏"
  excerpt: "Dedicate all my life to my love"
  ---
  
  <div class="tiles">
  {% for post in site.posts %}
  	{% if post.categories contains 'game' %}
  		{% include post-grid.html %}
  	{% endif %}
  {% endfor %}
  </div><!-- /.tiles -->
  ````

  ![picture0](https://huskytgame.github.io/images/in-post/others/2019-09-06-StartHere/screenshot000.png)

  Step3：打开*_data*文件夹下的*navigation.yml*（导航文件）。如下添加代码：
  
  ````
  - title: Game
    url: /game/
    excerpt: "游戏"
    image: navigation/GameImg.jpg
  ````

  其中``image: navigation/GameImg.jpg``指向的是类别的预览图片的位置。
  
  Step4：在指定位置创建预览图片
  
  Step5：在*images/in-post/*文件夹下创建game文件夹，以后可以将game类别的文章的图片存放在此文件夹内。
  
- 2.提交文章或图片后，博客长时间不更新或不显示文章图片的情况，很大程度是*navigation.yml*（导航文件）中的代码编写出错，请详细检查。