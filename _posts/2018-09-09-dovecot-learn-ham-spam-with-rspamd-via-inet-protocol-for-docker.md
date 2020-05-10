---
layout: post
title: Dovecot learn ham/spam with rspamd via inet protocol for docker
date: 2018-09-09 16:28:01 +02:00
categories: SoWhatIsTheSolution
tags: [Linux, Dovecot 2.3.2.1-r3, rspamd 1.7.6]
description: Switch rspam learn ham/spam to inet protocol
fullview: true
author:
  name: Christian
  email: webmaster@christiansaga.de
---

I started rebuilding my mail server following [Thomas Leisters Howto](https://thomas-leister.de/mailserver-debian-stretch/). However I decided to dockerize the whole setup. With that I needed to get rid of any socket communication and move to tcp based communication between different docker containers.

This was surprisingly easy, as most components already communicate via tcp. However the learn spam and ham mechanism still uses a socket.
So here are some details for my setup:

* I used a user defined network via docker compose to connect the different containers. By that I have full control over the containers IPs
* Each process is running in one container, so I have unbound, redis, rspamd, dovecot, postfix
* Host system is a debian stretch
* Docker containers are based on Debian:stable-slim

**EDIT 14.04.20**: I switched my setup to Debian based Docker. Especially the postfix container needs to be NOT Alpine right now. The resolver implementation in musl-libc cripples the DNSSEC calls of postfix, making outgoing DANE unusable.

### So what is the solution
**BEWARE**: I am basing my guide on Thomas config linked above.

First you need to change a few details in the ham/spam piping.
Within the ```dovecot.conf``` down at the plugin settings you need to set the ```sieve_pipe_bin_dir``` option to the location, where the pipe scripts (following steps) will be stored. Beware to set the path as it will be in your docker image.
My setting: ```sieve_pipe_bin_dir = /usr/local/sbin```

Next adapt the sieve scripts. These scripts trigger the learning as you can see in ```dovecot.conf```. Ham on copying out of SPAM folder, Spam on copying into SPAM folder.
Do not forget to call ```sievec``` after placing them in the sieve folder.

learn-spam.sieve
{% highlight conf%}
require ["vnd.dovecot.pipe", "copy", "imapsieve"];
pipe :copy "rspamd-pipe-spam";
{% endhighlight %}

learn-ham.sieve
{% highlight conf%}
require ["vnd.dovecot.pipe", "copy", "imapsieve"];
pipe :copy "rspamd-pipe-ham";
{% endhighlight %}

Now adapt the pipe scripts itself. These scripts will actually connect to rspamd to deliver the mail for learning.
During docker image creation you will need to copy the ```rspamd-pipe-spam``` and ```rspam-pipe-ham``` scripts into the ```sieve_pipe_bin_dir``` location (first step ) and make them executable.
The script is connecting via the container name ```rspamd``` if you have a different one, you need to change or use the IP.

rspamd-pipe-spam
{% highlight bash%}
#!/bin/bash
cat $1 | /usr/bin/curl -s --data-binary @- http://rspamd:11334/learnspam
exit 0
{% endhighlight %}

rspamd-pipe-ham
{% highlight bash%}
#!/bin/bash
cat $1 | /usr/bin/curl -s --data-binary @- http://rspamd:11334/learnham
exit 0
{% endhighlight %}

To allow this scripts to call rspamd you need to allow the IP of dovecot for the worker controller.

worker-controller.inc
{% highlight conf%}
bind_socket = "rspamd container>:11334";
password = "<your pwd as described in the guide>";
secure_ip = "<dovecot container ip>";
{% endhighlight %}

This should enable ham/spam learning via sieve within a docker setup.

####Addendum
To train existing mails, e.g. from an old server, you need to execute the following commands in the dovecot docker. Please make sure you adapt paths, if you changed them.
Learn HAM: ```find /var/vmail/mailboxes/*/*/mail/cur -type f -exec /usr/local/sbin/rspamd-pipe-ham {} \;```
Learn SPAM: ```find /var/vmail/mailboxes/*/*/mail/Spam/cur -type f -exec /usr/local/sbin/rspamd-pipe-spam {} \;```
