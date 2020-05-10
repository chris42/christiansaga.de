---
layout: post
title: RSyslog simple sorting to files, here iptables
date: 2015-04-27 17:17:00 +02:00
categories: SoWhatIsTheSolution
tags: [Linux, iptables, Logrotate, RSyslog]
description: Divert iptables log to own log file
fullview: true
author:
  name: Christian
  email: webmaster@christiansaga.de
---

If you have a firewall, you probably have 99% of the entires in your syslog filled with iptables denied info. So there are two options to get rid of that. Turn of the logging of iptables or move it out of the syslog.

As I want to have the logging, I need to move it. You will find a lot of explanations on RSyslog in the [official documentation](http://www.rsyslog.com/doc/master/index.html), however I found that the simple case I was looking for was not described.

I am using the following to create the syslog entries:

{% highlight bash%}
iptables -I INPUT 5 -m limit --limit 5/min -j LOG --log-prefix "iptables denied: " --log-level 7
{% endhighlight %}

### So what is the solution
You might have more logging or similar ones in your configuration. The important part here is the ```--log-prefix``` as we will sort via this.

To create a filter, create ```/etc/rsyslog.d/iptables.conf``` and add the following

{% highlight conf%}
# Proper file permissions
$FileCreateMode 0600
# Ruleset for sorting
if $msg contains 'iptables denied: ' then {
    # new destination
    /var/log/iptables.log
    # stop processing message, otherwise it will carbon to syslog
    stop
}
{% endhighlight %}

After restarting rsyslogd, this will now divert the iptables entries to the new logging location identified by the prefix.

The ```stop``` is particularly important, as the rule above diverts the logging, but without the ```stop``` it would still end up in syslog. If you google setups like this you will often find a ```~``` instead of the ```stop```. Which is deprecated and cause a warning in the log, however works as well.

**Sidenote**: Remember to put the ```stop``` in brackets for the if/then clause, otherwise you stop all logging to syslog.

As a last step you should create a logrotation for the new log. I did it very simple by creating the file ```/etc/logrotate.d/iptables``` and added

{% highlight conf%}
/var/log/iptables.log {
  rotate 5
  monthly
  compress
  missingok
  notifempty
}
{% endhighlight %}

Logrotate is started via cron and will read the new config in the next run. Alternatively you can run the config directly via ```logrotate -f /etc/logrotate.d/iptables```
