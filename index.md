---
layout: default
title: yayun's blog 
tagline:
---
{% include JB/setup %}
<!--{% assign first_page=site.posts[1] %}
{{ first_page.content }}-->
{% for post in site.posts  limit:3  %}
---
###[{{post.title}}]({{post.url}})
{{post.content | truncatewords:35}}    

[Read more...]({{post.url}})

{% endfor%}
---
##Recent posts
---
<ul class="posts">
  {% for post in site.posts  limit:10  %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

