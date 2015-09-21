---
layout: page-full-width
title: My Writing
permalink: /writing/
---

{% for piece in site.pieces %}
<h6>
	<a href="{{piece.url}}">{{piece.name}}</a>
	<span class="label label-default">{{piece.date}} </span>
</h6>
{% endfor %}
