---
layout: page
title: Things that happen while coding.
tagline: by Roman
---
{% include JB/setup %}

I started this page to write down some of my experiences with the Pebble watch and iOS SDK. Let´s look where this is going.

<ul class="posts">
{% for post in site.posts limit: 5 %}
  <div class="post_info">
    <li>
	    <a href="{{ post.url }}">{{ post.title }}</a>
	    <span>({{ post.date | date:"%Y-%m-%d" }})</span>
    </li>
    </br> <em>{{ post.excerpt }} </em>
    </div>
  {% endfor %}
</ul>


