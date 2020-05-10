---
layout: post
title: Unlock LUKS via SSH in Debian
date: 2016-03-20 01:56:15 +02:00
categories: SoWhatIsTheSolution
tags: [Linux, Debian Jessie]
description: Configure Debian to offer SSH login to unlock LUKS encryption
fullview: true
author:
  name: Christian
  email: webmaster@christiansaga.de
---

As already described in my previous post [Headless Debian install via SSH](sowhatisthesolution/2016/03/13/headless-debian-install-via-ssh), I am dealing with a headless system. As I am encrypting my system and drives with LUKS, I need a way to enter the password in case of a reboot.

### So what is the solution
First install Dropbear on the **server** by ```apt-get install dropbear```. Then configure initramfs network usage; edit ```/etc/initramfs-tools/initramfs.conf```. You probably have to add the lines for dropbear and update the device string.
This configuration is using DHCP to obtain an IP, if you have a static configuration, use: ```IP=<SERVER-IP>::<STANDARD-GATEWAY>:<SUBNETMASK>:<HOSTNAME>:eth0:off```

{% highlight conf%}
#
# DROPBEAR: [ y | n ]
#
# Use dropbear if available.
#
DROPBEAR=y

DEVICE=eth0
IP=:::::eth0:dhcp
{% endhighlight %}

Next, delete the standard private and public keys on the **server**

{% highlight bash%}
rm /etc/initramfs-tools/root/.ssh/id_rsa
rm /etc/initramfs-tools/root/.ssh/id_rsa.pub
{% endhighlight %}

Then create your own key pair (we assume you use id_rsa as a name) on your **client machine** and upload it to the server.

{% highlight bash%}
ssh-keygen
scp ~/.ssh/id_rsa.pub myuser@debian_headless:id_rsa.pub
{% endhighlight %}

After that, log in to the **server** and add the key to authorized_key file an remove the public key on the **server**.

{% highlight bash%}
ssh myuser@debian_headless
sudo sh -c "cat id_rsa.pub &gt;&gt; /etc/initramfs-tools/root/.ssh/authorized_keys"
rm id_rsa.pub
{% endhighlight %}

Now we need to update initramfs and grub by ```update-initramfs -u -k all``` and ```update-grub2```

On some configurations the network won't get reconfigured on runtime values, hence we need to trigger an update. Edit ```/etc/network/interfaces``` and add as first line of the primary interface ```pre-up ip addr flush dev eth0```

Restart server and log in from your **client** with ```ssh -i ~/.ssh/id_rsa root@<server-ip>``` to set the password to unlock

{% highlight bash%}
echo -n "<LUKS encryption password>" > /lib/cryptsetup/passfifo
exit
{% endhighlight %}
**EDIT**: on newer systems a ```cryptroot-unlock``` will suffice.

The server should now boot normally and regular SSH should come up.

#### Optional
You can also create a little script for the passphrase in ```/etc/initramfs-tools/hooks/unlock```

{% highlight bash%}
#!/bin/bash
PREREQ=""
prereqs() {
  echo "$PREREQ"
}

case $1 in
prereqs)
prereqs
exit 0
;;
esac

. /usr/share/initramfs-tools/hook-functions

cat > "${DESTDIR}/root/unlock" << EOF #!/bin/sh /lib/cryptsetup/askpass 'passphrase: ' > /lib/cryptsetup/passfifo
EOF

chmod u+x "${DESTDIR}/root/unlock"

exit 0
{% endhighlight %}

Do not forget to make it executable with ```chmod +x /etc/initramfs-tools/hooks/unlock``` and update initramfs with ```update-initramfs -u -k all``` and ```update-grub2```
