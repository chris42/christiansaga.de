---
layout: post
title: Deactivate bluetooth on startup
date: 2020-04-19 12:00:00 +0200
categories: SoWhatIsTheSolution
tags: Linux
fullview: true
description: Keep bluetooth from activating on startup
author:
  name: Christian
  email: webmaster@christiansaga.de
---

For some reason my laptop thinks bluetooth is the technology to go and activates it on startup of Linux Mint/Ubuntu.

Searching the internet you will find a lot of answers for this problem, that quite brutally kill the bluetooth service or block devices. All not very elegant.

### So what is the solution

Most laptops use [tlp](https://linrunner.de/tlp/) for power management. If you don’t you should check it out. With that it is quite easy. Uncomment and edit in ```/etc/tlp.conf:```

{% highlight conf %}
DEVICES_TO_DISABLE_ON_STARTUP="bluetooth"
{% endhighlight %}

You can also edit tlp to remember the last state when powering down. But for me deactivating bluetooth was all enough.
