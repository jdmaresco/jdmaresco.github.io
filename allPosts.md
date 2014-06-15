---
layout: page
title: All Posts
permalink: /all-posts/
---

<ul class="posts">
	{% for post in site.posts %}
  		<li>
    		<span class="post-date">{{ post.date | date: "%m/%d/%y" }}</span> 
    		<a class="post-link" href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
  		</li>
	{% endfor %}
</ul> 