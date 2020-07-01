---
layout: post
title: What I learnt using docker as a beginner
date: 2020-06-01 00:00:00 +02:00
categories: HowTo
tags: [Linux, Docker]
fullview: true
description: Sort of beginners best practices, or things I learnt using docker
author:
  name: Christian
  email: webmaster@christiansaga.de
published: true
---
Running my own server setup with docker, I ran into some issues over time. To not forget what I did and maybe to help others here is what I found.

It is a gathering of simple things I now try to include in all my docker setups, as they make life with docker easier. For some this might be obvious things. If you have additional thoughts, I am happy to hear of them.

#### Do not mix your hosts file system with your docker mounts
It might be a tempting idea to store config files of your container within your regular /etc or logs to /var/log. However you easily get lost to know which file is used by whom. Log files may get merged with others and so on. Hence have your own directory in which you store your docker stuff. It also makes backups, git use etc. easier.

#### Use git - a lot
You should use git to version everything you do on docker. Starting with your dockerfiles, to configuration files for services. Especially building a new container needs a lot of testing. Things might go forth and back, hence a history and versioning is utterly helpful.

#### Keep ENTRYPOINT and CMD separate
Use the entrypoint for something that needs to be done before each start of the service, like correcting permissions, update small things, etc.
Keep the command separated from this (hence do not put the service start command in the entrypoint) so you can override it easily.

#### Use dumb-init in your own Dockerfiles
It is a small package you can easily install and easily add to your image. It simply puts a tiny init system into the container to properly start and stop your service. You can read more on the [dumb-init github page](https://github.com/Yelp/dumb-init).
A lot of images out there ignore this. It is not per se harmful, but a lot of services get killed instead of properly shut down.

Prosody container example:

{% highlight docker %}
ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["bash", "-c", "/usr/local/sbin/entrypoint.sh && exec prosodyctl start"]
{% endhighlight %}

#### Learn about IPv6, iptables and internal docker networks
Docker takes over iptables when running a container, be sure to understand what is happening. Especially take a look on ```internal``` networks, as they can easily secure database container and other backend services.

For IPv6 better still use NAT. I didn't like that in the beginning as well, however you run into so many troubles (like dynamic prefixes, limited IPv6 configuration options, etc.), that it is just way easier to use NAT again. Sadly docker does not support NAT on IPv6, for that use [https://github.com/robbertkl/docker-ipv6nat](https://github.com/robbertkl/docker-ipv6nat). It is a simple binary to run in the background, that fills the gap.

#### Make sure your container runs with the correct user:group
Especially when using bind-mounts, make sure your use proper UID:GID combinations. I have created the users on my host machine that will be used inside the container and pass the UID:GID into the container. This ensures, that files stored on the host will not by accident be accessible by other processes due to UID overlap.

Again a prosody example on debian for your Dockerfile

{% highlight docker %}
ARG uid=1001
ARG gid=1002

# Add user and group before install, for correct file permission
RUN addgroup --gid $gid prosody \
    && adduser --system --disabled-login --disabled-password --home /var/lib/prosody --uid $uid --gid $gid prosody
{% endhighlight %}

#### Use docker logging
It is tempting to just mount a folder and store logfiles. However when using images, e.g. from hub.docker it can be quite difficult to correctly map this and often leads into some sort of chaos.

I actually like the webserver approach to just link the logfiles to ```/dev/stdout``` and ```/dev/stderr```

{% highlight docker %}
RUN ln -sf /dev/stdout /var/log/nginx/access.log
RUN ln -sf /dev/stderr /var/log/nginx/error.log
{% endhighlight %}

With that you can use the regular ```docker logs``` command and use configuration options to divert it to whatever log facility you are using.

#### Make sure your container are in the right timezone
A lot of images nowadays use the ```TZ=<timezone>``` as an environment variable in the container to control the timezone. You should set this always and use it in your own Dockerfiles

{% highlight docker %}
RUN ln -sf /usr/share/zoneinfo/$TZ /etc/localtime
{% endhighlight %}

Nothing is more confusing than log files with non matching timestamps.

#### If you need to build from source use multi-stage builds
Instead of bloating your production image with build-tools and sources, use [multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/).

With that you can create a build image, run it and use the build results in a regular image:

Shortened example for coturn

{% highlight docker %}
### 1. stage: create build image
FROM debian:stable-slim AS coturn-build

# Install build dependencies
[...]

# Clone Coturn
WORKDIR /usr/local/src
RUN git clone https://github.com/coturn/coturn.git

# Build Coturn
WORKDIR /usr/local/src/coturn
RUN ./configure
RUN make


### 2. stage: create production image
FROM debian:stable-slim AS coturn
[...]

# Copy out of build environment
COPY --from=coturn-build /usr/local/src/coturn/bin/ /usr/local/bin/
COPY --from=coturn-build /usr/local/src/coturn/man/ /usr/local/man/
COPY --from=coturn-build /usr/local/src/coturn/sqlite/turndb /usr/local/var/lib/coturn/turndb
COPY --from=coturn-build /usr/local/src/coturn/turndb /usr/local/turndb

# Install missing software
[...]
{% endhighlight %}

#### Make sure your services start in order
A lot of errors come up when services that depend on each other just start in random order. Sometimes you are lucky and it works, sometimes you get an error.
Within docker-compose you can use the [depends_on](https://docs.docker.com/compose/compose-file/#depends_on) directive.
If you have for some reason separate setups, you can use the following bit within your entrypoint:

{% highlight bash %}
/bin/bash -c " \
  while ! nc -z mariadb 3306; \
  do \
    echo 'Waiting for MariaDB'; \
    sleep 5; \
  done; \
  echo 'Connected!';"
{% endhighlight %}
