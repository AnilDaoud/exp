---
layout: post
title:  "Hello World!"
date:   2016-04-30 10:38:27
tags: experiments firstpost jekyll
---

First post! I'll post random bits and pieces on this site.
It is powered by [Jekyll][jekyll] which I think will be just the right mix of user-friendly and nerd for me.

It allows for code snippets:

**Ruby**
{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

**Python**
{% highlight python %}
def snowclone(x):
    print(x, “ is the new black.”)
snowclone(“Python”)
#=> prints 'Python is the new black.' to STDOUT.
{% endhighlight %}

And I'm sure, many others.

It was ridiculously easy to setup, compared to bloated Wordpress.

{% highlight bash %}
# install jekyll
apt-get install jekyll
# create website
jekyll new ~/exp
cd ~/public_html
ln -s ~/exp/_site/ exp
ls ~/exp
   about.md     css/      _includes/  _layouts/  _site/
   _config.yml  feed.xml  index.html  _posts/
# Edit _config.yml and about.md.
# baseurl: no trailing slash and get rid of the url setting.
# When done
cd ~/exp
jekyll build
# and the result is this website!
{% endhighlight %}

[jekyll]:    http://jekyllrb.com
