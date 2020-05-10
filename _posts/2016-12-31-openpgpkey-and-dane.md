---
layout: post
title: OPENPGPKEY and DANE
date: 2016-12-31 14:31:20 +02:00
categories: SoWhatIsTheSolution
tags: Linux
description: Publish PGP keys via DANE
fullview: true
author:
  name: Christian
  email: webmaster@christiansaga.de
---

As a long time PGP user I wanted to improve the key landscape and offer my public key via DANE. This is quite simple, if you have a DNS host, that supports DNSSEC and the different needed DANE types. (Note: I use [core-networks.de](www.core-networks.de))

The principle behind this: You publish your PGP key in your signed DNS and by that a correctly configured mail system could opt into sending you encrypted E-Mails even without having exchanged keys before.

I have tried to use the ```gpg2``` function ```--print-dane-records``` (available from 2.1.9 upwards), however could not generate a usable data part of the DNS record via this.

### So what is the solution
As a prerequisite you need to have created your own PGP keys with the E-Mail you want to use as an User ID. Note: I am only doing a per E-Mail Setup.
Use the following website: [https://www.huque.com/bin/openpgpkey](https://www.huque.com/bin/openpgpkey)

Generate an OPENPGPKEY and the output pretty much does the trick.
The generated output is sorted as follows:

* **Owner Name** goes into the record. Be careful, if your DNS provider automatically adds the domain.If you used ```--print-dane-records``` you need to concatenate the ID of the key (before TYPE61) and string after $ORIGIN
The syntax is ```<SHA256 hash of your e-mail name before the @>._openpgpkey.<your domain>```

* Out of **Generated DNS OPENPGPKEY Record** you take the part within the ```()``` into the data part of the record. Here you need to transform the key into one line string.
If you used ```--print-dane-records```, I discarded the data part, as I could not get it to work. Simply use your key data exported with ASCII armor (```-a``` parameter) without the last line. Of course leaving headers and footers out as well.

* The type is OPENPGPKEY

* Class is IN

* TTL you can set on a decent value. For testing I used a hour.

To test the setup use

{% highlight bash%}
dig OPENPGPKEY <owner name goes here>
{% endhighlight %}

This will simply send back the data block and test the DNS setup.
To test the DANE lookup do the following

{% highlight bash%}
gpg2 --auto-key-locate clear,dane,local -v --locate-key <your e-mail goes here>
{% endhighlight %}

This gives quite a good feedback on the setup and tells you if the key was fetched via DANE or not.
