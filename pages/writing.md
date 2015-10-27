---
layout: page-full-width
title: My Writing
permalink: /writing/
---

{% for piece in site.pieces %}
  <p>
	  <a href="{{piece.url}}">{{piece.name}}</a>
  	<span class="label label-default">{{piece.date}} </span>
	</p>
{% endfor %}
