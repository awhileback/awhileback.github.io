---
layout: default
---

{% for post in site.posts limit: 10 %}
<h1 align="center"><a href="{{ post.url }}">{{ post.title }}</a></h1>
<br/>
<h3 align="center"> <i>&bull; {{ post.date | date: "%b %-d, %Y" }}</i></h3>
<br/>
<hr>
<br/>
{% endfor %}