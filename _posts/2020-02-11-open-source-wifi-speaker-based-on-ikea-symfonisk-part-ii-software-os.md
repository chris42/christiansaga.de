---
layout: post
title: 'Open Source Wifi Speaker based on IKEA Symfonisk – Part II: Software / OS'
date: 2020-02-11 15:28:17 +01:00
categories: DIY
tags: [Linux, ALSA, crossover, DIY, equalizer, Hifiberry, piCorePlayer-v5.0.0, Raspberry Pi, Sonos, Symfonisk, Wifi]
description: Add open software to Raspbian built wifi speaker
fullview: true
author:
  name: Christian
  email: webmaster@christiansaga.de
---

After finishing the [hardware in part I of this little tutorial](/diy/2020/02/08/open-source-wifi-speaker-based-on-ikea-symfonisk-part-i-hardware.html), we need to set up the operating system to run our speakers. Of course it should be open source software.

There are multiple solutions out there to do multiroom streaming, however the [slimserver or squeezserver](https://github.com/Logitech/slimserver) eco system looks the easiest for me right now. It seems to support everything I am looking for:

* Central streaming server with lightweight clients for playback
* Multiroom and sync features
* Extensible with plugins
* Internet radio
* Streaming services, like Spotify or Tidal
* Open source

Most of the features are actually done in the server and are pretty easy to set up. I am running the server in a docker on my NAS, where it also has access to my local music library.

This part however is focusing on the player and have that as slim and robust as possible. I checked a setup on [HifiberryOS](https://www.hifiberry.com/hifiberryos), [Rasbian](http://www.gerrelt.nl/RaspberryPi/wordpress/tutorial-installing-squeezelite-player-on-raspbian/), etc.. However was convinced to start with [piCorePlayer](https://www.picoreplayer.org/) as a clever solution based on [tinycore Linux](http://www.tinycorelinux.net/).

The advantage of piCorePlayer / tinycore Linux is, that it runs completely in RAM. Hence you can unplug the speaker without too much worries about the SD card in the raspi. Also the system is very small. I use now only a 1GB SD card, which is not even used 10% (Sadly you can not buy such small cards anymore ;-)).
I could have used piCore (the barebone tinycore Linux for raspis), however then I would need to do a lot of work that has already been done by the piCorePlayer team.

So now we only need to extend the piCorePlayer with specialties of our setup:

* [Richard Taylor's LADSPA plugins for Active Loudspeakers](http://faculty.tru.ca/rtaylor/rt-plugins/) to create the crossover and mono downmix of the speaker
* Upfront Wifi configuration to avoid using a network cable

### Step 1: Prepare Richard Taylor's LADSPA plugins
As tinycore is loading everything into RAM on boot, the system is build out of different extensions, which are pretty much zips with the files for that extension. All loaded extensions get merged into one filesystem in RAM on which the system is running. Most things are already prepackaged and can be downloaded as an extension, however within the repositories, I could not find the LADSPA plugins of Richard as a prepared extension. However creating such extension is not too complicated.

[Download Plugins]({{ site.baseurl }}/assets/download/rt-plugins-0.0.6.zip)

If you want to compile the plugins yourself, it makes it easier to use a piCore setup on the Raspberry.

### Step 2: Get piCorePlayer and install it to the SD card of your speaker
Get the latest piCorePlayer [here](https://www.picoreplayer.org/main_downloads.shtml) and write it to your SD card.

Next you need to add your wifi information to have it log in automatically.
For that, mount the just created SD card. It should have two partitions, a PCP_BOOT (probably mmcblk0p1) and PCP_ROOT (probably mmcblk0p2) partition.
Create a wpa_supplicant.conf in the root of the PCP_BOOT partition and fill in the relevant fields (country, SSID, password). There is also a sample file which you can just copy, rename and edit. This is also explained [here](https://www.picoreplayer.org/how_to_setup_wifi_on_pcp_without_ethernet.shtml).

Next you need to resize the root partition to use the free space on the SD card. On my systems (Linux Mint), I can easily to this with gparted (right click on partition to resize). Otherwise you can do it in the piCorePlayer GUI after booting. However to do it right away is easier, as we want to copy Richard Taylors plugins as well.
The plugins from Step 1, will now be added to the piCorePlayer by copying them to the PCP_ROOT storage

{% highlight bash%}
cp rt-plugins-0.0.6.tcz /mnt/mmcblk0p2/tce/optional/
cp rt-plugins-0.0.6.tcz.md5.txt /mnt/mmcblk0p2/tce/optional/
echo "rt-plugins-0.0.6.tcz" >> /mnt/mmcblk0p2/tce/onboot.lst
{% endhighlight %}

The last line adds the extension to the boot list, so they will be loaded and available in the file system after boot.
Now the piCorePlayer is ready to boot and to run the speaker.

### Step 3: Configure piCorePlayer
You need to do some basic configuration of piCorePlayer to run it with the speaker. For that use a browser to access the piCorePlayer web interface under the IP of our speaker.
First activate the advanced mode on the tab of the bottom of the ```Main Page```. During the configuration you will have a few reboots. I would do each one of them, to be thorough.

If you haven't extended the filesystem yet, you need to do it now (```Main Page``` -> ```Resize FS```) and probably still need to copy the plugin

After that update the player under ```Main Page``` -> ```Full Update```. Update the extension under ```Main Page``` -> ```Update```. Check for possible hotfixes under ```Main Page``` -> ```HotFix```.

Then head to the ```Squeezelite Settings``` page and set Amp2 as audio output device. After saving and possible reboot, use the ```Card control``` button and deactivate the raspis internal audio device down at the bottom. Only after that squeezelite will start.
Back on the ```Squeezelite Settings``` page you have to enter the name of this player and the IP of the LMS server.
Finally on the ```Tweaks``` page enable the ```ALSA 10 band Equalizer```. Probably another reboot is helpful and our basic configuration is complete.

The speaker now will be working, however it will sound weird. You still need to activate the crossover and probably use the equalizer.

### Step 4: Activate crossover, mono downmix and equalizer settings
For the crossover you need to connect to the player console via SSH. After login you need to edit the ```/etc/asound.conf``` file. Vi is installed in the system.
You will find a pre-generated configuration from piCorePlayer. Edit this to match the following:

{% highlight conf%}
# default - Generated by piCorePlayer
pcm.!default {
  type plug
  slave.pcm "plugequal"
}

ctl.!default {
  type hw
  card 0
}

pcm.pcpinput {
  type plug
  slave.pcm "hw:1,0"
}

#---ALSA EQ Below--------
ctl.equal {
  type equal;
  controls "/home/tc/.alsaequal.bin"
  library "/usr/local/lib/ladspa/caps.so"
}

pcm.plugequal {
  type equal;
  slave.pcm "plug:crossover";
  controls "/home/tc/.alsaequal.bin"
  library "/usr/local/lib/ladspa/caps.so"
}

pcm.equal {
  type plug;
  slave.pcm plugequal;
}

#--- Speaker crossover below-----
pcm.crossover {
  type ladspa
  slave.pcm "amp2"
  path "/usr/local/lib/ladspa"
  channels 6
  plugins
  {
    0 {
       label RTlr4lowpass # lowpass left output to channel 2
       policy none
       input.bindings.0 "Input"
       output.bindings.2 "Output"
       input { controls [ 3000 ] }
      }
    1 {                                                    
       label RTlr4lowpass # lowpass right output to channel 3
       policy none                       
       input.bindings.1 "Input"                            
       output.bindings.3 "Output"                           
       input { controls [ 3000 ] }       
      }                                       
    2 {                                                     
       label RTlr4hipass # highpass left output to channel 4
       policy none                       
       input.bindings.0 "Input"                             
       output.bindings.4 "Output"                           
       input { controls [ 3000 ] }                         
      }                                       
    3 {                                                     
       label RTlr4hipass # highpass right output to channel 5
       policy none                                         
       input.bindings.1 "Input"                            
       output.bindings.5 "Output"                           
       input { controls [ 3000 ] }                          
      }                                  
  }                                                         
}                                

# --- Output device and mono downmix below -----                                                            
pcm.amp2 {                                                  
  type plug                                                 
  slave {                                                   
    pcm "t-table"                
    channels 6                                              
    rate "unchanged"                                       
  }                                                         
}                               
                                 
pcm.t-table {                                               
  type route                                               
  slave {                                                  
    pcm "hw:0,0"                                            
    channels 2                                              
  }                              
  ttable {                       
    2.0 0.5 # Mix both stereo low (main) to mono            
    3.0 0.5                                                 
    4.1 0.5 # Mix both stereo high (tweeter) to mono        
    5.1 0.5                                                
  }                                                         
}                                                  
                                               
pcm.plughw.slave.rate = "unchanged";
{% endhighlight %}

So what is this doing? Basically it tells ALSA to use a chain of filters for the default device and then send it to the hardware:

* pcm.!default: redirect the default device to plugequal
* pcm.plugequal/pcm.equal: this is the ALSA equalizer and first step in our chain. We will configure it later. We can use this to influence the sound of our speaker
* pcm.crossover: here we use Richard Taylor's plugins to split our stereo signal in 4 channels. The highpass ones above 3000Hz and the lowpass below that
* pcm.amp2/pcm.t-table: this is the hardware device for playback (hw:0,0), the table does the mono downmix for high- and lowpass channels

After this, we get 2 mono signals, one above 3000Hz, one below, each left and right of the stereo signal downmixed. These are then put to the tweeter and mid-range speaker. Having the equalizer in the chain, we additionally have the opportunity to tune the sound of our speaker. This will impact all sound being send to the speaker.

Save the file and get back to the command line. Next you need to set the equalizer

### Step 5: Setting the equalizer
The ALSA equalizer is set via the command line.

{% highlight conf%}
export TERM=xterm
sudo alsamixer -D equal
{% endhighlight %}

The first line will prevent the error ```Error opening terminal: xterm-256color```.

You will get a nice looking console equalizer, in which you can tune the sound of the speaker (left/right, up/down arrow keys).
Now this is complicated, as sound is something very personal. Hence my settings might not be the right ones for you. I advise to come back later when you play some music and tune this to your taste (I am happy to hear of any settings). Changes to the equalizer are audible right away.

Here are my settings (observe the numbers on the bottom)

![Equalizer](/assets/media/equalizer.png){:width="100%"}

As you can see, I did not change too much. As far as I know, within an equalizer the less is better.

**IMPORTANT step**: We now edited some files on the player, to make that survive the next reboot, we need to backup our config.
Go back to the piCorePlayer web interface to the ```Main Page``` and click the ```Backup``` button. Now the settings will survive possible reboots.

### Step 5: Add controls to the front buttons
To make our buttons on the front work during playback, we need another extension.

Head to ```Main Page``` -> ```Extensions``` and click on the ```Available``` tab.
Under the ```Available extensions``` select ```pcp-sbpd.tcz``` and click load. This installs all necessary plugins to monitor the raspi pins.

Next - via SSH - create the file ```/home/tc/sbpd-script.sh``` with the following content (please adjust the ```-A``` and ```-P``` to your LMS server instance and port. You can try auto discovery by removing both flags as well):

{% highlight bash%}
#!/bin/sh

# start pigpiod daemon
sudo pigpiod -t 0

# give the daemon a moment to start up before issuing the sbpd command
sleep 1

# load uinput module, then set the permission to group writable, so you don't need to run sbpd with root
sudo modprobe uinput
sudo chmod g+w /dev/uinput

# issue the sbpd command
sbpd -s -d -A 192.168.2.9 -P 9000 b,11,PLAY,2,0 b,10,VOL-,2,0 b,9,VOL+,2,0
{% endhighlight %}

After saving the file, make it executable ```chmod +x /home/tc/sbpd-script.sh```
This script basically starts the GPIO monitoring. The last line defines the GPIOs to watch. here it is 9, 10 and 11 (each right after the ```b,```).
If you remember the table out of the hardware wiring for the buttons this is SW_Mute, SW_Down, SW_Up. If you changed the pins used on the raspi, then you will need to adapt it here.

If you are interested in the whole syntax given there or to add a long press event, use ```sbpd --help``` on the command line to get the details.

Now the script needs to be launched at startup. In the web interface go to ```Tweaks```and add ```/home/tc/sbpd-script.sh``` as ```User command #1``` and click Save.

Now finally go back to the ```Main Page``` and hit ```Backup``` one last time.
One last reboot and your speaker should be ready to rock...

### Addendum: Changing crossover frequency and equalizer
The selected crossover frequency and equalizer settings are based on my personal taste. It might be that there are better options. Overall, this is not a high audiophile level of speaker, hence do not expect too much.
If you want to fiddle with that, the following commands might help a bit.

The crossover frequency is set in ```/etc/asound.conf```. I guess you can spot the 3000. ;-). The frequencies for each filter should match, otherwise it sounds quite weird, but feel free to play around with it. Just beware, that forcing a speaker into frequencies it was not build for is stressful for it and might even break it. Hence not change your tweeter into a sub-woofer and vice versa.

Calling the equalizer:
```export TERM=xterm
sudo alsamixer -D equal```

Relaod ALSA after changes (reboot is always better):
```alsactl kill rescan```

Reduce the volume directly on the speaker to 70% (otherwise the next command is very loud):
```amixer sset Digital 70%```

Produce a pink noise speaker test sound (for frequency measurements):
```speaker-test -c2 -tpink```
