---
layout: post
title: Duplicity backup to Amazon S3 on Debian
date: 2015-04-27 16:36:46 +02:00
categories: SoWhatIsTheSolution
tags: [Linux, Boto 2.38.0, Boto 2.40.0-1, Debian Jessie, Debian Wheezy, Duplicity 0.7.02-1, Duplicity 0.7.06-2~bpo8+1, Duply 1.9.1-1]
fullview: true
description: S3 error due to old duplicity versions in Debian Jessie
author:
  name: Christian
  email: webmaster@christiansaga.de
---

I am using duplicity with duply to do PGP encrypted backups to Amazon S3. As there are enough guides around for the basic setup (i used [Backup unter Linux mit duply](https://www.thomas-krenn.com/de/wiki/Backup_unter_Linux_mit_duply">Backup unter Linux mit duply)), I won't go into details of that.

However if you are planning to use the new EU (Frankfurt) location called eu-central-1 you will run into problems as this location only supports the [V4 authentication](https://stackoverflow.com/questions/26533245/the-authorization-mechanism-you-have-provided-is-not-supported-please-use-aws4).
You might encounter the following error when using duply (even if the config worked on other locations before)

{% highlight log%}
The authorization mechanism you have provided is not supported. Please use AWS4-HMAC-SHA256
{% endhighlight %}

### So what is the solution
This is a bit tricky, as the Debian packages are not up to date and you need to mix backports and unstable versions.
To solve this you will need to upgrade your duplicity and boto framework packages as following:

* **duplicity** needs version 0.7.02-1
This version is available within the ```backport``` branch of Debian. I used package pinning to get this package. You can easily find articles for this via Google, I found [APT Pinning](https://unix.stackexchange.com/questions/107689/how-to-install-a-single-jessie-package-on-wheezy) helpful.
Be careful to understand pinning, as you might confuse APT with a wrong configuration. You should always use ```sudo apt-cache policy [packagename]``` and the ```-s``` parameter on upgrades first to check what will happen.
Currently I am using duplicity 0.7.06-1~bpo8+1.

* **python-boto** needs version 2.36.0
Previously this was only available via PyPi, however a newer version is now available in the ```unstable``` branch. Hence again you can install this via pinning.
Currently I am using python-boto 2.40.0-1

* **duply** did not need additional changes, I am using version 1.9.1-1 out of the regular stable branch.
