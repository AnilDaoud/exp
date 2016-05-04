---
layout: post
title:  "Pimping Jekyll"
date:   2016-05-2 10:38:27
tags: Jekyll
---

So, three posts in three days. So far I love Jekyll, it is easy to customize. To wit, here are a few examples of what you can tweak.

## Pimping the front page

Thanks [[Jekyll doc]](https://jekyllrb.com/docs/posts/)

I put this in index.html at the root of the website, to have a list of posts and a quick summary with it. post.excerpt rules.

{% highlight html %}
{% raw %}
{% for post in site.posts %}
      <li>
        <span class="post-date">{{ post.date | date: "%b %-d, %Y" }}</span>
        <a class="post-link" href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
        <p>{{ post.excerpt | remove: '<p>' | remove: '</p>'}}</p>
      </li>
{% endfor %}
{% endraw %}
{% endhighlight %}

If your website isn’t auto-watched/auto-hooked etc, run
{% highlight bash %}
jekyll build
{% endhighlight %}
Profit!

## Pimping the header

Thanks [[Joshua Lande]](http://joshualande.com/jekyll-github-pages-poole)

- Create a new page called archive.md in the site's directory
{% highlight html %}
{% raw %}
---
layout: page
title: Archive
permalink: /Archive/
---

## Blog Posts

{% for post in site.posts %}
  * {{ post.date | date_to_string }} &raquo; [ {{ post.title }} ]({{ post.url | prepend: site.baseurl }})
{% endfor %}
{% endraw %}
{% endhighlight %}
- modify header.html with 
{% highlight html %}
{% raw %}
<a class="page-link" href="{{ "/feed.xml" | prepend: site.baseurl }}">Feed</a>
{% endraw %}
{% endhighlight %}
- If your website isn't auto-watched/auto-hooked etc, run 
{% highlight bash %} jekyll build {% endhighlight %}
- Profit !

## Pimping the footer

Thanks [[David Elbe]](http://david.elbe.me/jekyll/2015/06/20/how-to-link-to-next-and-previous-post-with-jekyll.html)

- Google how to add _Next post_, _Previous post_, to posts
- Copy paste this snippet at the end of post.html
{% highlight html %}
{% raw %}
<div style="display:block; width:auto; overflow:hidden;">
 {% if page.previous.url %}
   <a style="display:block; float:left; width:50%; text-align:left;" href="{{page.previous.url | prepend: site.baseurl}}">« {{page.previous.title}}</a>
 {% else %}
<p style="display:block; float:left; width:50%;">First post</p>
{% endif %} 
 {% if page.next.url %}
  <a style="display:block; float:left; width:50%; text-align:right;" href="{{page.next.url | prepend: site.baseurl}}">{{page.next.title}} &raquo;</a>
 {% endif %}
</div>
{% endraw %}
{% endhighlight %}
- If your website isn't auto-watched/auto-hooked etc, run 
{% highlight bash %} jekyll build {% endhighlight %}
- Profit !

## Pimping statistics

Thanks [[Joshua Lande]](http://joshualande.com/jekyll-github-pages-poole)

One might wonder why I would want official confirmation that I'm the only one reading this website but hey

- Suscribe to [Google Analytics](https://analytics.google.com)
- Google will provide you with a snippet of code
- Create a file in _include, named google_analytics.html and the code from google
{% highlight html %}
<script>
  (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
  (i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
  m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
  })(window,document,'script','https://www.google-analytics.com/analytics.js','ga');
  ga('create', 'XXXXXXXXXXX', 'auto');
  ga('send', 'pageview');
</script>
{% endhighlight %}

- in _layouts/defaults.html, add
{% highlight html %}
{% raw %}
 {% include google_analytics.html %}
{% endraw %}
{% endhighlight %}
- If your website isn't auto-watched/auto-hooked etc, run 
{% highlight bash %} jekyll build {% endhighlight %}
- Profit!

## Pimping comments

Thanks [[Steel]](http://steelx.github.io/best-internet-tips/2014/11/23/Add-google-plus-comments-box-to-jekyll-website.html)

What a blog without a comment section? Are you sure? why?
Again since I'm the only one reading, it is useless, but again i refer you to [this](http://anil.diwi.org/stuff/extraits/gauthier.txt)

- Create _include/comments.html with the following code
{% highlight html %}
{% raw %}
<script src="https://apis.google.com/js/plusone.js"></script>
  <div class="g-comments"
      data-href="{{site.baseurl}}{{page.url}}"
      data-width="642"
      data-first_party_property="BLOGGER"
      data-view_type="FILTERED_POSTMOD">
  </div>
{% endraw %}
{% endhighlight %}
- Include this in the post.html file in _layout
{% highlight html %}
{% raw %}
	{% include comments.html %}
{% endraw %}
{% endhighlight %}
- If your website isn't auto-watched/auto-hooked etc, run 
{% highlight bash %} jekyll build {% endhighlight %}
- Profit!


