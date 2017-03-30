--- 
layout: default 
title: Blog archive 
permalink: /archive/ 
---
<div id="archive">
  {% for post in site.posts  %}
    {% capture this_year %}{{ post.date | date: "%Y" }}{% endcapture %}
    {% capture next_year %}{{ post.previous.date | date: "%Y" }}{% endcapture %}
    {% if forloop.first %}
    <div class="archive__block">
      <h2 class="archive__year">{{this_year}}</h2>
      <ul class="archive-list"> 
      {% endif %}
       <li><a href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a> <span class="archive__post__date">{{ post.date | date: "%b %-d" }}</span></li>
        {% if forloop.last %}
      </ul>
    </div>
    {% else %}
    {% if this_year != next_year %}
    </ul></div>
      <div class="archive__block">
       <h2 class="archive__year" id="{{ next_year }}-ref">{{next_year}}</h2>
      <ul class="archive-list">
    {% endif %}
    {% endif %}
  {% endfor %}
</div>