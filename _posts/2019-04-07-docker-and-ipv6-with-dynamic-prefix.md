---
layout: post
title: Docker and IPv6 with dynamic prefix
date: 2019-04-07 00:02:59 +02:00
categories: SoWhatIsTheSolution
tags: [Linux, Debian Stretch, Docker, dynamic prefix, ip6tables, IPv6, ndppd, SLAAC]
description: Use Docker with a dynamic prefix setup
fullview: true
author:
  name: Christian
  email: webmaster@christiansaga.de
---

Getting IPv6 on your private connection should be quite easy by today. Getting services you might have hosted before via dynDNS towards IPv6 however is a bit more work. Mainly due to the fascinating concept of dynamic prefixes.

Most private dial up connections with dual-stack use dynamic prefixes. This gives you three options:

* Change provider or ask yours nicely for a fixed prefix (nice, but not possible for everyone)
* Use NAT and IPv6 ULAs with tools like these: [https://github.com/robbertkl/docker-ipv6nat](https://github.com/robbertkl/docker-ipv6nat9) (working, but pretty much the old IPv4 world)
* What we want to do: or improvise, adapt, overcome.

The article is about how to work with a dynamic prefix and not use something like DHCPv6 and basically results in a few problems with each prefix change triggered via SLAAC.

* You need IPv6 dynDNS to update your AAAA records
* Docker will need updates for the network configurations
* Neighbor solicitation will need to be updated
* ip6tables becomes more complicated

### So what is the solution
**BEWARE**: This is not about configuring your router. I assume that it is not blocking IPv6 traffic IN- or OUTBOUND.

First you need to get your docker IPv6 setup working with the current prefix unchanged. I used the official Docker documentation here [https://docs.docker.com/v17.09/engine/userguide/networking/default_network/ipv6/#how-ipv6-works-on-docker](https://docs.docker.com/v17.09/engine/userguide/networking/default_network/ipv6/#how-ipv6-works-on-docker)

This is very straightforward. But some comments to this:
* Docker itself advises against using the default network and advises to setup and use your own network. This is what I did, however I would advise to also setup the default network properly, as containers started without any network will come up in that
* I got a /56 prefix from my provider, so best for me was a /80 subnet for docker. This would allow each docker container to code their MAC into the IPv6
* If you are not sure, how to cut your networks, use a IPv6 calculator from the internet, it helps to visualize. I also decided to number my networks manually to make them more readable for me (hence using the MAC only for random containers)
* While setting this up, make sure your ip6tables is in ACCEPT mode on your host machine, especially in the FORWARD chain, as this is the relevant chain for DOCKER. The additional docker chains you might know from IPv4 are not existent in IPv6

I will use the following format to reference the IPv6 (letters the prefix, numbers my subnet numbering): ```aaaa:bbbb:cccc:dddd:1::/80```
Yes, my network is just ```1```, so containers then get ```::1::1```, ```::1::2``` etc. Keep it simple here.

Next, add the ndppd daemon to automate the neighbor solicitation on the host machine. It is available as a package in Debian. You will need this, as otherwise more scripting is needed.
I used the following config (/etc/ndppd.conf):

{% highlight conf%}
# route-ttl  (NEW)
# This tells 'ndppd' how often to reload the route file /proc/net/ipv6_route.
# Default value is '30000' (30 seconds).

route-ttl 30000

# proxy
# This sets up a listener, that will listen for any Neighbor Solicitation
# messages, and respond to them according to a set of rules (see below).
#  is required. You may have several 'proxy' sections.

proxy eth0 {

   # router <yes|no|true|false>
   # This option turns on or off the router flag for Neighbor Advertisement
   # messages. Default value is 'true'.

   router no

   # timeout
   # Controls how long to wait for a Neighbor Advertisment message before
   # invalidating the entry, in milliseconds. Default value is '500'.

   timeout 500

   # ttl
   # Controls how long a valid or invalid entry remains in the cache, in
   # milliseconds. Default value is '30000' (30 seconds).

   ttl 30000

   # rule [/]
   # This is a rule that the target address is to match against. If no netmask
   # is provided, /128 is assumed. You may have several rule sections, and the
   # addresses may or may not overlap.

   rule aaaa:bbbb:cccc:dddd:1::/80 {
      # Only one of 'static', 'auto' and 'interface' may be specified. Please
      # read 'ndppd.conf' manpage for details about the methods below.

      # 'auto' should work in most cases.

      # static (NEW)
      # 'ndppd' will immediately answer any Neighbor Solicitation Messages
      # (if they match the IP rule).

      # iface
      # 'ndppd' will forward the Neighbor Solicitation Message through the
      # specified interface - and only respond if a matching Neighbor
      # Advertisement Message is received.

      # auto (NEW)
      # Same as above, but instead of manually specifying the outgoing
      # interface, 'ndppd' will check for a matching route in /proc/net/ipv6_route.

      auto

      # Note that before version 0.2.2 of 'ndppd', if you didn't choose a
      # method, it defaulted to 'static'. For compatibility reasons we choose
      # to keep this behavior - for now (it may be removed in a future version).
   }
}
{% endhighlight %}

**eth0** represents the host interface in which requests will show up. Hence this is the interface towards your router.
If your docker container has the global routeable IPv6 ```aaaa:bbbb:cccc:dddd:1::1``` and a request from the outside is delivered to your router (identified by prefix), your router will ask every connected node who has this IPv6. However the container is not connected to the router but to the docker host. Hence the docker host needs to answer to the router and then forward it to the docker container. For this the NDP proxying is needed to listen on eth0 on the host machine.

**rule** is the docker network you want to activate in your docker host, so it can answer to the router. You can have multiple rule sections for multiple docker networks.
With the ndppd deamon you can skip the manual adding as described in the documentation.

**Info:** If you use ```ip -6 neigh``` to chech if it was registered, know that the daemon will only add it when traffic is routed. Hence trigger it with something like a ping.
At this point you should be able to have your docker containers on IPv6 and ping6 the internet.

Next, to prepare for prefix changes, we need a script that updates several things. Sadly I could not find a hook that would allow to trigger right when the prefix changes. the dhcpclient apparently has a hook, but as we are not using DHCP...
Hence the script is triggered by cron every 5 minutes and checks for a prefix change.
Below find an example, however you will need to adapt it to your needs, especially networks after the prefix, the docker compose and container information and dyndns setup.

update-prefix.sh
{% highlight bash %}
#!/bin/bash
# Update script to adapt docker networking to changed IPv6 prefix

export LC_ALL=C

DYN_USER="<your dyndns user>"
DYN_PASS="<your dyndns pass>"

# Grep current configured prefix from docker settings
PREFIX_OLD=$(grep -o -P '(?<=fixed-cidr-v6": ").*(?=:0::)' /etc/docker/daemon.json)

# Get latest prefix from ip, latest = highest ttl, hence on top
PREFIX_NEW=$(ip -6 addr show eth0 | grep inet6 | grep -v 'inet6 f[de]' | awk '{print $2}' | cut -f 1-4 -d : | head -n 1)

# If prefix changed, update docker settings and restart docker service
if [ $PREFIX_OLD != $PREFIX_NEW ]; then
    echo "Prefix needs update"

    echo "   Stopping Docker container..."
    /usr/local/bin/docker-compose -f docker-compose.yml down
    /usr/bin/docker network rm docker-network

    echo "   Changing prefix from ${PREFIX_OLD} to ${PREFIX_NEW}..."    
    sed -i 's'"@$PREFIX_OLD"'@'"$PREFIX_NEW"'@g' /etc/docker/daemon.json
    sed -i 's'"@$PREFIX_OLD"'@'"$PREFIX_NEW"'@g' /etc/ndppd.conf

    echo "   Restarting Docker..."
    /etc/init.d/docker restart
    /usr/bin/docker network create \
        --driver=bridge \
        --subnet=192.168.1.0/24 \
        --gateway=192.168.1.1 \
        --ipv6 \
        --subnet="${PREFIX_NEW}:1::/80" \
        docker-network
    /usr/local/bin/docker-compose -f docker-compose.yml up -d

    echo "   Restarting ndppd..."
    /etc/init.d/ndppd restart

    echo "   Updating DNS..."
    IP_NEW=$(/usr/bin/docker inspect -f '{{ "{{range .NetworkSettings.Networks"}}}}{{ "{{.GlobalIPv6Address"}}}}{{ "{{end"}}}}' nginx)
    curl -4 -s -u $DYN_USER:$DYN_PASS "<dyndns update url>"

DYN_USER=""
DYN_PASS=""

fi

exit 0
{% endhighlight %}

The script basically does the following:
* extracts the prefix stored in the ```daemon.conf``` of docker for the default network and the newest prefix of eth0, here between ```fixed-cidr-v6": ``` and ```:0::``` in the config file. I used ```0``` for my default docker network (hence ```aaaa:bbbb:cccc:dddd:0::/80```) and ```1``` for my custom network.
* compares both and only acts if there is a change
* updates the new prefix in ```/etc/docker/daemon.conf``` and ```/etc/ndppd.conf```
* restarts docker and ndppd for changes to take effect
* extracts new IPv6 from container named ```nginx``` and updates dyndns with this IPv6

Lastly adapt ip6tables to changing IPv6 prefix is not easy, as it seems, that the function is undocumented.

This is only needed, if you actually DROP all forwarding rules in your ip6tables. Hence would close off all docker containers from the internet, other than exceptions. I would recommend this, as in IPv4 Docker the standard configuration would only open exposed ports to the internet. With IPv6 you have to take care of the filtering yourself. And remember -with this config- all containers have internet routable IPv6s.

To expose a Docker container (e.g. here ```aaaa:bbbb:cccc:dddd:1::1```) with port 80 to the internet, you normally would add following exception to your host forwarding.

{% highlight bash%}
ip6tables -I FORWARD -p tcp -m tcp -d aaaa:bbbb:cccc:dddd:1::1 --dport 80 -j ACCEPT
{% endhighlight %}

However as the prefix ```aaaa:bbbb:cccc:dddd``` could change you need something different:

{% highlight bash%}
ip6tables -I FORWARD -p tcp -m tcp -d ::1:0:0:1/::ffff:ffff:ffff:ffff --dport 80 -j ACCEPT
{% endhighlight %}

As you can see, the prefix is gone and you are able to target a specific container. Now you only need to allow inter-container traffic, this should be handled by the following.
{% highlight bash%}
ip6tables -I FORWARD -s ::1:0:0:1/::ffff:ffff:ffff:ffff -j ACCEPT
{% endhighlight %}

You could add these rules in the end of the script. I fixed the container IPv6 ends in the docker-compose, hence I can leave them static in the firewall.

#### Addendum
I would recommend to test each of this part by part, e.g. make sure, the dyndns works before putting the script together, etc. It makes debugging and changes way easier.
Thanks to input of following sources:
* [https://docs.docker.com/v17.09/engine/userguide/networking/default_network/ipv6/#how-ipv6-works-on-docker](https://docs.docker.com/v17.09/engine/userguide/networking/default_network/ipv6/#how-ipv6-works-on-docker)
* [https://github.com/DanielAdolfsson/ndppd](https://github.com/DanielAdolfsson/ndppd)
* [http://blog.dupondje.be/?p=17](http://blog.dupondje.be/?p=17)
* [https://www.puzzle.ch/de/blog/articles/2017/06/13/docker-container-mit-ipv6-anbinden](https://www.puzzle.ch/de/blog/articles/2017/06/13/docker-container-mit-ipv6-anbinden)
