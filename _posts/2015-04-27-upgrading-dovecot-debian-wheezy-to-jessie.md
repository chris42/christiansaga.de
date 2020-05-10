---
layout: post
title: Upgrading Dovecot Debian Wheezy to Jessie
date: 2015-04-27 14:55:38 +02:00
categories: SoWhatIsTheSolution
tags: [Jessie upgrade, Linux, Dovecot 2.2.13-11, Systemd]
description: Socket vs. service issues during upgrade of Dovecot
fullview: true
author:
  name: Christian
  email: webmaster@christiansaga.de
---

The automatic upgrade is pretty straight forward and you probably need only to check the configs for changes and merge them. For me dovecot came up after the restart and was doing its work. However I received the following errors in ```/var/log/mail.err```

{% highlight log%}
dovecot: master: Error: systemd listens on port 143, but it's not configured in Dovecot. Closing.
dovecot: master: Error: systemd listens on port 993, but it's not configured in Dovecot. Closing.
dovecot: master: Error: systemd listens on port 993, but it's not configured in Dovecot. Closing.
{% endhighlight %}

Looking for this in the net, I could only find a maintainer discussion about socket vs. service configuration and what happens if both is set? What config should have precedence (the config actually).

That pointed me to the socket configuration within Systemd. Indeed within the update the socket unit file was activated (in ```/etc/systemd/system/sockets.target.wants```).

### So what is the solution
To get rid of the error, just deactivate the dovecot.socket via ```systemctl disable dovecot.socket```.
