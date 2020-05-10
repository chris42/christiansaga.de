---
layout: post
title: Upgrading SQLGrey Debian Wheezy to Jessie
date: 2015-04-27 15:29:11 +02:00
categories: SoWhatIsTheSolution
tags: [Jessie upgrade, Linux, SQLGrey 1.8.0-1, Systemd]
description: SQLGrey start before database after upgrade
fullview: true
author:
  name: Christian
  email: webmaster@christiansaga.de
---
SQLGrey itself will not be updated during the upgrade towards Jessie, however the change to Systemd will result in a stub automatically created by Systemd around the original init.
The problem I encountered was, that SQLGrey would start before my database (MySQL). Hence you receive something like the following in ```/var/log/syslog```

{% highlight log%}
sqlgrey: dbaccess: can't connect to DB: Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock’
{% endhighlight %}

The problem here is, that the generated stub is not obeying the init scripts dependencies and not having any Systemd ones, hence the services are starting in a "wild" order.

### So what is the solution
We need to create at least the SQLGrey unit file and add the dependency into it.
Create ```/etc/systemd/system/sqlgrey.service``` and add:

{% highlight conf%}
[Unit]
Description=SQLgrey Postfix Grey-listing Policy service
After=syslog.target network.target mysql.service
[Service]
Type=forking
PIDFile=/var/run/sqlgrey.pid
ExecStart=/usr/sbin/sqlgrey -d
[Install]
WantedBy=multi-user.target
{% endhighlight %}

Note the ```mysql.service``` in the ```After``` line. This is the needed dependency.

As you might have noticed, there is no ```mysql.service``` unit file existing, as the package maintainers have deprioritized to create one (there are two bugs [#765425](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=765425) and [#742900](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=742900).

However you can reference the stubs as well.
If you need to adapt this for other services: To find the name of the stub you can use ```systemd-analyze blame```. It gives you a list of the services started. Details you can get with ```systemctl```, e.g. ```systemctl status mysql.service```. The details will tell you in the ```Loaded``` line if it is a real unit file or just a init script stub.
