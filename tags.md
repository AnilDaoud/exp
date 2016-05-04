---
layout: page
title: Tag cloud
permalink: /tags/
---

<div class="home">
  <p class="post-meta" style="text-align: justify;">
     {% capture site_tags %}{% for tag in site.tags %}{{ tag | first }}{% unless forloop.last %},{% endunless %}{% endfor %}{% endcapture %}
     {% assign tags = site_tags | split:',' | sort %}
     {% include tagcloud.html %}
  </p>
</div>
