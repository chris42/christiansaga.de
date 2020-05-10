---
layout: post
title: Headless Debian install via SSH
date: 2016-03-13 12:00:00 +02:00
categories: SoWhatIsTheSolution
tags: [Linux, Debian]
description: Install Debian without display
fullview: true
author:
  name: Christian
  email: webmaster@christiansaga.de
---

Having build my own NAS system recently, I realised I do not have any monitors or keyboards at home anymore. Hence installing Debian will be hard.
I looked around and the solution would be a headless install via ssh.

This post is based on some work from S.G. Vulcan's post [Installing Debian using only SSH](http://www.sgvulcan.com/2010/01/06/installing-debian-using-only-ssh/)
His post was a good start, but I only could make it work for a Debian Jessie netinstall image after some changes.

### So what is the solution
Download the latest netinstall image from Debian, I used ```debian-8.3.0-amd64-netinst.iso```

Mount the ISO to a folder

{% highlight bash%}
mkdir isoorig
sudo mount -o loop -t iso9660 debian-8.3.0-amd64-netinst.iso isoorig
{% endhighlight %}

Copy to new folder called isonew

{% highlight bash%}
mkdir isonew
rsync -a -H --exclude=TRANS.TBL isoorig/ isonew/
{% endhighlight %}

Change the menu to load SSH on boot by default, edit ```isonew/isolinux/txt.cfg``` remove (if existing) ```menu default``` from ```label install``` and add:

{% highlight conf%}
label netinstall
  menu label ^Install Over SSH
  menu default
  kernel /install.amd/vmlinuz
  append auto=true vga=788 file=/cdrom/preseed.cfg initrd=/install.arm/initrd.gz locale=en_US console-keymaps-at/keymap=us

default netinstall
{% endhighlight %}

Create ```isonew/preseed.cfg``` file. I adapted the locale and keyboard settings for Germany and added the selection of the keyboard-configuration. This would otherwise be an open question during the install and we won't reach the SSH startup.

Also I added a check for non-free firmware, which popped up on one of my machines which had wireless.

{% highlight conf%}
#### Contents of the preconfiguration file
### Localization
# Locale sets language and country.
d-i debian-installer/locale select de_DE
# Keyboard selection.
d-i console-keymaps-at/keymap select de
d-i keyboard-configuration/xkb-keymap select de
### Network configuration
# netcfg will choose an interface that has link if possible. This makes it
# skip displaying a list if there is more than one interface.
d-i netcfg/choose_interface select auto
# Any hostname and domain names assigned from dhcp take precedence over
# values set here. However, setting the values still prevents the questions
# from being shown, even if values come from dhcp.
d-i netcfg/get_hostname string newdebian
d-i netcfg/get_domain string local
# If non-free firmware is needed for the network or other hardware, you can
# configure the installer to always try to load it, without prompting. Or
# change to false to disable asking.
d-i hw-detect/load_firmware boolean true
# The wacky dhcp hostname that some ISPs use as a password of sorts.
#d-i netcfg/dhcp_hostname string radish
d-i preseed/early_command string anna-install network-console
# Setup ssh password
d-i network-console/password password install
d-i network-console/password-again password install
{% endhighlight %}

Recreate the ```isonew/md5sum.txt```, it is read only, so you need to change this. Also I had better luck with creating the ```md5sum.txt``` with the changed commands below.

{% highlight bash%}
chmod 666 md5sum.txt
find -follow -type f -exec md5sum {} \; > md5sum.txt
chmod 444 md5sum.txt
{% endhighlight %}

Create ISO file to burn with xorriso. If you do not have it installed use ```apt-get install xorriso```.

{% highlight bash%}
xorriso -as mkisofs -D -r -J -joliet-long -l -V "Debian headless" -b isolinux/isolinux.bin -c isolinux/boot.cat -iso-level 3 -no-emul-boot -partition_offset 16 -boot-load-size 4 -boot-info-table -isohybrid-mbr /usr/lib/syslinux/isohdpfx.bin -o ../debian-8.3.0-amd64-netinst-headless.iso ../isonew
{% endhighlight %}

xorriso is creating a correct partition table, which is for some reason not done with mkisofs only. The original command would work in VMs, maybe even on a cd-rom, however not for USB sticks.

The ISO can be burned to an USB stick and used to boot. It will automatically configure the network with DHCP (yes, you need to have a way to find the IP, e.g. on your router) and start SSH.
The user for the ssh connection is ```installer``` the password is ```install```.
