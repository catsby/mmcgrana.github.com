---
layout: post
title: noprocrast(1)
---

# {{page.title}}

<span class="meta">March 3 2010</span>

Inspired by [Alex's post](http://al3x.net/2009/09/14/my-get-back-to-work-hack.html), I wrote a command line script called `noprocrast`. This script updates `/etc/hosts` to make distracting websites unreachable from my machine:

{% highlight sh %}
sudo cp /etc/noprocrast_hosts /etc/hosts
{% endhighlight %}

The corresponding `/etc/noprocrast_hosts`:

{% highlight sh %}
127.0.0.1 localhost db001 db002 db003 db004 web030 web048
255.255.255.255 broadcasthost
::1             localhost
fe80::1%lo0     localhost

127.0.0.1 news.ycombinator.com
127.0.0.1 reddit.com www.reddit.com
127.0.0.1 reader.google.com
127.0.0.1 nytimes.com www.nytimes.com
{% endhighlight %}

In case I really need to visit one of these sites, I use the following `procrast` script:

{% highlight sh %}
sudo cp /etc/procrast_hosts /etc/hosts
echo `date` >> ~/.procrasts
{% endhighlight %}

Then if I want to see how I have been doing with procrastination, I `tail ~/.procrasts`.

The `/etc/procrast_hosts` file is just:

{% highlight sh %}
127.0.0.1 localhost db001 db002 db003 db004 web030 web048
255.255.255.255 broadcasthost
::1             localhost
fe80::1%lo0     localhost
{% endhighlight %}

If you haven't tried an `/etc/hosts` hack like this, I suggest giving it a try. You may be surprised by how much time you save.
