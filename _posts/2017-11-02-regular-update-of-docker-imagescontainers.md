---
layout: post
title: Regular update of Docker images/containers
date: 2017-11-02 22:06:12 +02:00
categories: SoWhatIsTheSolution
tags: [Linux, Debian Stretch, Docker 17.05.0-ce]
description: Make sure all docker containers are updated regularly
fullview: true
author:
  name: Christian
  email: webmaster@christiansaga.de
---

After converting my servers into docker setups, I was in need to update the images/containers regularly for security reasons. Baffled I found, that there is no standard update method to make sure that everything is up-to-date.
The ephemeral setup allows you to throw away your containers and images and recreate them with the latest version. As easy as this sounds, you figure that there are some loopholes in the setup.

First we need to understand, there are 3 types of images that we need to keep up-to-date
* Images from the docker hub, that just get pulled and are used as they are with some configs
* Images from the docker hub, that get pulled and then are only used as base for own dockerfiles
* The images created out of own dockerfiles

Ridiculously all three of them need to be updated to make sure everything is up-to-date (most important, a new local build won't get the latest base image update) and additionally we have to care for cleanup.

I found some solutions in the net to automatically update docker, the so far best version by [binfalse.de](https://binfalse.de/2017/01/24/automatically-update-docker-images). But this leaves out my own dockerfiles with a build and some minor steps, pruning, etc. So I am only using the dupdate script out of the [Handy Docker Tools](https://binfalse.de/2016/12/03/handy-docker-tools/#dupdate-updates-images) to incorporate in a little script.

### So what is the solution
**WARNING**: This just updates images. If your setup needs additional update steps, you need to plan these in. Otherwise you risk breaking your setup.

Multiple steps are needed to completely update.
First, I use ```/usr/local/sbin/dupdate -v``` to update all docker images coming from a hub, covering the ones I use directly and as base for builds.

This will give an error for the images you created out of your own dockerfiles, but update all pulled ones from docker hub.
Second, I update the images of my own dockerfiles by rebuilding all via ```/usr/local/bin/docker-compose -f docker-compose.yml build --no-cache```. If you use docker without docker-compose, you just have to do something similar for each dockerfile.

This will use the newly pulled base images in their build, hence create the latest version for your dockerfile.
**IMPORTANT**: The ```--no-cache``` is needed to force the update of self build Dockerfiles. Docker determines an update on your own builds only by the commands in the Dockerfile (hence if they changed). It CANNOT see a version change of a package installed with e.g. ```apt-get install```. But exactly that version change you want, so you have to force a rebuild.

Now the images are all updated and we only need to restart the containers.

#### Addition for cleaning up
However you end up with a lot of images tagged or named <none>. These are your old images, which are now cluttering the hard drive.
The ones only tagged with <none> are the ones you updated from docker hub, the ones with name and tag <none> are the ones you build.
You will need to run ```/usr/bin/docker image prune -a --force``` to get rid of them and free up space.
Warning: This will erase all older images. If you need them as a safety precaution, skip this step.
