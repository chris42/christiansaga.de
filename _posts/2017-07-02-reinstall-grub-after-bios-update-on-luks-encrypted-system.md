---
layout: post
title: Reinstall GRUB after BIOS update on LUKS encrypted system
date: 2017-07-02 22:28:04 +02:00
categories: SoWhatIsTheSolution
tags: [Linux, BIOS Update, GRUB, Linux, LUKS, UEFI]
description: Rescue LUKS setup after corruption by BIOS update
fullview: true
author:
  name: Christian
  email: webmaster@christiansaga.de
---

Due to reasons unknown I upgraded my Lenovos BIOS via a USB Stick. :-) Everything went well, however after reboot, all my boot options for Linux Mint were gone.
Turns out, somehow my boot setup was erased as well.
Using UEFI without CSM and without secure boot with LUKS encrypted Linux Mint, this was already an issue when first installing. Getting everything right seems to be more of good luck.

This answer ist mainly for straight forward installations of Ubuntu/Mint with LVM on LUKS and unencrypted boot. If you have a different setup, make sure to adapt the different mounts.

**Advice on using boot-repair utility:** Don't. The tool messes with a lot of configs unnecessarily. If you are not absolutely sure, what it does, don't use it.
As an example for myself: When using it to restore Grub, it edited my fstab and uncommented my root mapper and set it to noauto.
Result: You end up after Grub in an initramfs prompt, as the volume has not been unlocked. Possibly wasting time by checking on why LVM times out, cryptsetup not working, etc.

### So what is the solution
To get back your boot menu, I tried several things, e.g. boot-repair. However since my system is LUKS encrypted, I guess the tools all had some problems.
To get back my system, I accessed my old system via chroot from a Linux Live CD. In this case Linux Mint Live CD.

First boot from Linux Live CD, get keyboard locale and network set up. Unlock your LUKS device. Make sure, that the name in the end (```sda3_crypt```) is as specified in your original ```/etc/crypttab``` (yes, if you do not know, you need to open the crypt device somewhere, take a look, close and reopen it). Otherwise you might get a warning later on.

If you have a different setup, make sure to take the correct device.

{% highlight bash%}
cryptsetup luksOpen /dev/sda3 sda3_crypt
{% endhighlight %}

For overall LUKS informations and commands, take a look here: [https://wiki.ubuntuusers.de/LUKS/](https://wiki.ubuntuusers.de/LUKS/)

Next mount all necessary partitions out of the old system
To find the LUKS drives, use ```sudo lvscan```

For me my ```/root``` turns out to be in ```/dev/mint-vg/root```

If you have separated partitions, e.g. for home, or other devices, make sure to adapt parts below for mounting.

{% highlight bash%}
sudo mount /dev/mint-vg/root /mnt
sudo mount /dev/sda2 /mnt/boot
for i in /dev /dev/pts /proc /sys /run ; do sudo mount -B $i /mnt$i ; done
sudo mount -o bind /etc/resolv.conf /mnt/etc/resolv.conf
sudo mount /dev/sda1 /mnt/boot/efi/
{% endhighlight %}

Make sure you add the ```/mnt/boot/efi```, otherwise grub will complain ```grub-install: "cannot find EFI directory"```. It is not included in your boot partition, but a separate partition.

After that, enter the chroot environment with ```sudo chroot /mnt /bin/bash```

To install GRUB run ```sudo grub-install /dev/sda```

Now just a reboot is needed and the system should work as before.

#### Addendum
If you are here because something on boot is not working and you messed things up even further as I did, additionally ```update-initramfs -c -k all``` could help.
