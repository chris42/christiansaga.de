---
layout: post
title: SendXMPP mail forward on Debian Jessie
date: 2016-04-09 00:39:46 +02:00
categories: SoWhatIsTheSolution
tags: [Linux, Debian Jessie, Sendxmpp 1.23-1.1]
description: Forward system E-Mails via XMPP
fullview: true
author:
  name: Christian
  email: webmaster@christiansaga.de
---

To have a more comfortable way of receiving messages by my servers, I wanted all my root E-Mails to be forwarded to my mobile via XMPP. I only have a limited exim4 on my machine running, configured for local mail delivery only.

### So what is the solution
First install sendxmpp with ```apt-get install sendxmpp```, then create the config file as ```/etc/sendxmpp.conf``` and insert XMPP credentials:

{% highlight conf%}
<sender>@<sender_server>:<port> <password>
{% endhighlight %}

Set the right permissions ```chmod 600 /etc/sendxmpp.conf``` and owner ```chown Debian-exim:Debian-exim /etc/sendxmpp.conf```

Then create a script to call sendxmpp ```/usr/sbin/mail2xmpp```. It might be that you could put this completely into the alias, however I decided to use the script.
Exchange your receiving ID. ```-t``` enables the TLS connection for sending the message.

{% highlight conf%}
#!/bin/bash
echo "$(cat)" | sendxmpp -t -f /etc/sendxmpp.conf <receiver>@<receiving_server>
{% endhighlight %}

Make the script executable ```chmod 755 /usr/sbin/mail2xmpp``` and create the alias for the user, which E-Mails you want to forward in ```/etc/aliases```:

{% highlight conf%}
# /etc/aliases
root:,|/usr/local/bin/mail2xmpp
{% endhighlight %}

To activate pipe forwarding we have to create ```/etc/exim4/exim4.conf.localmacros``` SoWhatIsTheSolution

{% highlight conf%}
SYSTEM_ALIASES_PIPE_TRANSPORT = address_pipe
{% endhighlight %}

After that run ```newaliases``` and ```service exim4 restart``` for config to take effect.
Now you should be able to test if it works, by simply sending a local test E-Mail to user ```root```.
