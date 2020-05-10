---
layout: post
title: Upgrading Spamassassin Debian Wheezy to Jessie
date: 2015-04-27 17:38:19 +02:00
categories: SoWhatIsTheSolution
tags: [Jessie upgrade, Linux, Spamassassin 3.4.0-6]
description: Spamassasin not added to startup service
fullview: true
author:
  name: Christian
  email: webmaster@christiansaga.de
---

After the upgrade of Wheezy to Jessie, Spamassassin is not added to the startup services. Hence if you were using it before in your mail setup, you will run into the following error in ```/var/log/syslog```

{% highlight log%}
spamc: connect to spamd on ::1 failed, retrying (#1 of 3): Connection refused
spamc: connect to spamd on 127.0.0.1 failed, retrying (#1 of 3): Connection refused
{% endhighlight %}

Debian has a similar bug filed ([#764438](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=764438)), even after a fresh reinstall.

### So what is the solution
Simply activate Spamassassin via ```systemctl enable spamassassin```
