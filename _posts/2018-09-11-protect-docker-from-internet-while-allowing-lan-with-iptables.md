---
layout: post
title: Protect docker from internet while allowing LAN with iptables
date: 2018-09-11 18:05:04 +02:00
categories: SoWhatIsTheSolution
tags: [Linux, Docker, iptables]
description: Correct docker iptables setup to prevent unintended open ports
fullview: true
author:
  name: Christian
  email: webmaster@christiansaga.de
---

So some of you might have noticed, that docker is setting up its own rules in iptables. However these rules reside mostly in the FORWARD chains and not in the INPUT.
Most users, that setup their small server or laptop use INPUT rules to secure it. This is allright, as normally a server and a laptop are standalone and no routers.
However with installing docker it becomes a router for the containers, hence docker uses the FORWARD chains. Knowing iptables means, that traffic gets sorted after PREROUTING into INPUT or FORWARD. Hence the INPUT rules are not affecting the FORWARD chain.

Docker manages the FORWARD chain (and even the nat table) itself to control access to the containers. Basically it setups NAT and the FORWARDING to the containers as soon as you use the ```--port``` command or docker-compose ```port:``` setting.

In this docker does not differ between traffic from the internet or a local LAN. Everything that arrives at the FORWARD chain is used.

**ISSUE**: In the scenario of of having a laptop or small home server, which is usually connected via local LAN to the internet, this could lead to opening services that should only run in LAN, to the internet. Only an additional firewall outside could prevent this.

### So what is the solution
To get back to a state where we can control opening services to the internet, you need four iptables rules. We will use the DOCKER-USER chain, which should be empty and is made by docker for user written rules within the docker setup. Therefore they will not be touched by docker.

Before starting you should make a list of your local networks, including the docker network (even the default network).

First stop docker from communicating.
{% highlight bash%}
iptables -A DOCKER-USER -j DROP
{% endhighlight %}

This is adding a rule, that is just stopping any docker communication. After this all containers will be offline for the internet or locally.

Next allow local networks to communicate.
{% highlight bash%}
iptables -I DOCKER-USER -s <docker-network> -j RETURN
iptables -I DOCKER-USER -s <LAN> -j RETURN
{% endhighlight %}

The first rule allows the docker containers to communicate with each other, hence you need to add your docker network her or have multiple rules if you have multiple networks (e.g., 172.18.0.0/24).
The second rule allows the LAN to communicate again, hence it should be something along 192.168.1.0/24.
After this your docker services will be available from the local network, however not from the internet (IP spoofing would work, but you can only handle that with different interfaces). If you do not want to publish services to the internet, you can stop here.

Finally allow outside communication to reach containers
{% highlight bash%}
iptables -I DOCKER-USER -p tcp --dport <target-port> -j RETURN
{% endhighlight %}

Put here the target port of the service on the docker container to allow traffic.
IMPORTANT: This happens after nat, so if you have docker do some portmapping, e.g., 8080 -> 80 you need to put the container port 80 here.
With this you can have your docker services accessible on the LAN and decide which to publish to the internet.

**Notes:**
We use ```-I``` to add these rules before the DROP rule that was added first.
```RETURN``` is used as the chain DOCKER-USER is in front of all other docker rules in the filter table. Hence we need to return to those.
