---
layout: post
title: 'Open Source Wifi Speaker based on IKEA Symfonisk - Part I: Hardware'
date: 2020-02-08 14:39:57 +01:00
categories: DIY
tags: [Linux, Hifiberry, Raspberry Pi, Sonos, Symfonisk, Wifi]
description: Build your own fully functional wifi speaker with free components
fullview: true
author:
  name: Christian
  email: webmaster@christiansaga.de
sitemap:
  priority: 0.7
---

For years I had Sonos speakers back at home to play my music and stream internet radio, spotify, etc.
With the announcement of Sonos to not support older devices, it became clear, that again perfectly fine hardware will become useless, just because of software decisions. For long I was annoyed with samba v1 on the speakers, which couldn't be updated due to limited space inside the flash memory. For adding Alexa and other new features apparently there was always space.

Hence I thought to build a speaker that is build with standard components (can be repaired if broken) and uses open source software. However if you look into the internet, there is tons of howtos on how to build your own speaker and speaker casing, even some awesome builds into old radios etc. I wanted to have the small devices with good sound like I had with my Sonos (not audiophile high def super sound).
The importance for me was design, to have a device that fits nicely into the living room ([high WAF](https://en.wikipedia.org/wiki/Wife_acceptance_factor)).

So basically I need to rebuild a Sonos speaker. However these are quite expensive to just tinker with. But lately IKEA got the Symfonisk speakers for 99€. Hence I got one and rebuild it into a complete open source Wifi enabled speaker.

This is probably not the cheapest way of getting a wifi speaker, but the result was worth it for me. If you sum up the costs you are in the range of a small Sonos speaker, however with the huge advantage of being free from a single vendor and having a maintainable device.

What did I use (For some parts there are other options, e.g. the amp or power supply, but "This is my design").
The links below are just for informational purposes, feel free to use your own supplier.

* [IKEA Symfonisk](https://www.ikea.com/de/de/p/symfonisk-regal-wifi-speaker-weiss-30435204/)
* [Mean Well HDR-60-15 power supply (DIN-rail) 15 V/DC 4 A 60 W](https://www.conrad.de/de/p/mean-well-hdr-60-15-hutschienen-netzteil-din-rail-15-v-dc-4-a-60-w-1-x-1894080.html)
* [Raspi Zero WH](https://www.berrybase.de/raspberry-pi-zero-wh)
* [Hifiberry Amp2](https://www.hifiberry.com/shop/boards/hifiberry-amp2/)
* Micro SD card (8GB is more than enough)
* [SD card slot extension](https://www.conrad.de/de/p/radxa-kabelsatz-1x-microsd-stecker-1x-sd-karten-slot-weiss-2143254.html)
* [Micro USB port](https://www.berrybase.de/computer/kabel-adapter/usb/micro-usb/kabel-usb-2.0-micro-b-buchse-zum-einbau-usb-2.0-micro-b-stecker-1-m)
* [FPC adapter with 10pins, 0,5mm](https://www.ebay.de/itm/FFC-Adapter-Platine-auf-RM-2-54mm-0-5-mm-10-60-Adern-Flachbandkabel-FPC/183927242351)
* Wires for power supply >1mm diameter
* Spacers for the raspberry board (M2, so they fit to the Hifiberry ones, I used 1 and 2 cm ones)
* Small cables

I measured the setup in terms of power consumption. It uses roughly 2,5W in standby. Compared with Sonos, that is quite good.

You personally need some really basic soldering expertise and good common sense ("Don't eat from the yellow snow", "Don't touch live wires"). I glued most of the parts into the chassis, however the raspi can be unscrewed.

**DISCLAIMER: I am not a technician or expert on this, and learned a lot by doing this project. If you rebuild this, you do it on your own risk.**

If you found any errors, I am happy to hear of them.
If you try this guide, start off by building it outside of the case to verify correct wiring and function of each part.

### Step 1: Disassembly
First you need to completely disassemble the Symfonisk speaker. A good guide you will find here: [Hacking the Sonos Ikea Symfonisk Into a High Quality Speaker Amp](https://makezine.com/2019/08/16/hacking-the-sonos-ikea-symfonisk-into-a-high-quality-speaker-amp/)
That guide is basically doing the reverse: Using the Sonos parts for a regular speaker. Also a good project and probably good to sell off the Sonos components. However goal of this tutorial is to get independent of a single supplier.
After the disassembly you pretty much will have a lot of Sonos spare parts and en empty housing for the speaker. For the build we will only use/reuse the following, so make sure you do not damage them in disassembly:

* Housing/Chassis
* Front part with both drivers
* Wiring towards the speakers
* Power plug and cable on the back (not the wiring inside)
* Buttons on front with electronics and the flex cable towards mainboard
* Screws, etc.

### Step 2: Power supply
I used a DIN-rail power supply, as they are very compact. The speakers inside the Symfonisk are a 4Ohm mid range and a 8Ohm Tweeter. Hifiberry recommends a 18V 60W ~3A power supply, if you connect the Amp2 to 4-8Ohm speakers.

Finding a 18V power supply in compact form was quite a hassle. The Mean Well HDR-60-15 actually is in the 15V category, but can be tuned to deliver up to 18V. Mean Well even has open housing power supplies, which are a few € cheaper, however they are not as compact as this one.
The power supply has a little screw next to the outputs to increase voltage output. It is quite clearly marked. You can turn it fully for the 18V.

Alternatively you can use an external power supply (Like old notebook supply), however then you will need new plug and wiring at the backside.

### Step 3: Add the Raspberry Pi and Hifiberry Amp2
The regular raspi zero W does not have the pin header soldered on, that is why I went for the zero WH. If you happen to have a W only, the only thing to do is to [solder the header on and test it (German)](http://raspberry.tips/raspberrypi-tutorials/raspi-zero-gpio-pins-aufloeten-und-testen-teil-1).

The Amp2 is just placed on top of the raspi:

![Amp2 on Raspberry](/assets/media/amp2-on-raspi.jpg){:width="60%"}

On the picture you can see I used a regular plug to power the Amp2 with raspi, mainly for testing.

### Step 4: Wiring the power supply
Make sure you use proper cables for the power supply, I used >1mm diameter full copper and used the original plug from the backside of the Symfonisk and soldered new cables on towards the power supply. +/- does not matter here, as it is AC.

The Amp2 powers the raspi, so you only need to connect the Amp2 to the power supply. Here it is important to correctly wire +/- with the Amp2 as the output is DC. The power supply has two outputs, it does not matter which to use.

### Step 5: Extend Raspberry Pi ports
As the SD cards in raspis easily get damaged, I decided I want to extend the sd card slot and make it accessible from the outside when the housing is closed.
Also I want to extend one USB port to the back of the housing to possibly add some hdd or to connect a usb network adapter (should I not use wifi anymore). You could build in the network adapter and glue it on the old port, however I felt this to be more flexible.
This is pretty straight forward, as you just connect the USB port plug and the SD-card extender on the raspi.

### Step 6: Connect the Sonos buttons
The Symfonisk has pretty little buttons and LEDs on the front and it would be nice to be able to still use them. The parts you pulled out in the disassembly (+ the cable):

![Amp2 on Raspberry](/assets/media/button-platine.jpg){:width="60%"}

The rubber parts are not needed for now, but we need to check the electronics. You have 4 LEDs and 3 buttons on it, all wired to a FPC 10 pin, 0,5mm connector. With the adapter linked above you can actually use the original cable and solder on some cables to connect it to the raspi.

![Amp2 on Raspberry](/assets/media/FPC-adapter.jpg){:width="60%"}

On the adapter you can see the lines of the cable in 1-10. With a multimeter you can figure out what it connects to on the button electronics board (or use my table below). We also need to map them to raspi pins to use them later on. The Amp2 uses HW pins 3, 5, 7, 12, 35, 38 and 40. Also 27 and 28 are off limits for the eeprom.

**Please be careful**: HW pins are not the GPIO numbers. See chart below, HW pins in the middle. Nr. 1 ist the squared one on the raspi and Amp2 board.

![Amp2 on Raspberry](/assets/media/raspberry-pi-gpio-belegung.png){:width="60%"}

To pull everything together, I did the following mapping for the buttons and LEDs for the adapter and raspi connection:

<div class="table-responsive">

<table>
  <thead>
    <tr>
      <th>CABLE/ADAPTER</th>
      <th>FUNCTION</th>
      <th>HW-PIN</th>
      <th>GPIO</th>
      <th>EXPLANATION</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td>SW_Mute</td>
      <td>23</td>
      <td>11</td>
      <td>play/pause button</td>
    </tr>
    <tr>
      <td>2</td>
      <td>SW_Mute</td>
      <td>23</td>
      <td>11</td>
      <td>just use one</td>
    </tr>
    <tr>
      <td>3</td>
      <td>Green LED</td>
      <td>16</td>
      <td>23</td>
      <td> </td>
    </tr>
    <tr>
      <td>4</td>
      <td>Red LED</td>
      <td>18</td>
      <td>24</td>
      <td> </td>
    </tr>
    <tr>
      <td>5</td>
      <td>GND</td>
      <td>20</td>
      <td>GND</td>
      <td>just use one</td>
    </tr>
    <tr>
      <td>6</td>
      <td>Amber LED</td>
      <td>22</td>
      <td>25</td>
      <td> </td>
    </tr>
    <tr>
      <td>7</td>
      <td>GND</td>
      <td>20</td>
      <td>GND</td>
      <td>just use one</td>
    </tr>
    <tr>
      <td>8</td>
      <td>White LED</td>
      <td>15</td>
      <td>22</td>
      <td> </td>
    </tr>
    <tr>
      <td>9</td>
      <td>SW_UP</td>
      <td>21</td>
      <td>9</td>
      <td>volume up button</td>
    </tr>
    <tr>
      <td>10</td>
      <td>SW_DW</td>
      <td>19</td>
      <td>10</td>
      <td>volume down button</td>
    </tr>
  </tbody>
</table>

</div>

When you solder everything together you get something like this.

![Amp2 on Raspberry](/assets/media/kabelbaum.jpg){:width="60%"}

You can now connect the Button panel with the original flexcable to this.

### Step 7: Wire the speakers
Polarity is key in here, hence make sure you connect it correctly. The speakers and the Amp2 are marked with +/-.

I used the Sonos wiring, but you can of course create new ones. On the Sonos wiring, the big plugs are the + (this is also marked on the speakers)
Yes, you see correctly, I am using one stereo channel for of the Amp2 for the mid range and one channel for the tweeter speaker. No hardware crossover, we will need to do that later on with ALSA.

**Now everything is ready. If you want to do some tests you should be able to. However be careful with live wires!**

### Step 8: Fit everything into the chassis
Now everything needs to go into the chassis. I mainly glued everything into it, but made sure, I can unscrew the raspi for possible maintenance. To hold it in place I used spacers which are then glued to the chassis.

![Amp2 on Raspberry](/assets/media/spacers-on-raspi.jpg){:width="60%"}

You can see the original speaker wiring in the foam, the red and white wires go to the power supply.
In the back you see the red white again, which is the wiring from the power supply to the plug on the back of the chassis.
The SD-card extension is leaving the raspi on the left, the USB port is the thick cable in the front.

The spacers are stepped, left 1cm, on the right 2cm. The chassis is not even in the back, hence you need this to fit. Only these parts are glued and the raspi is screwed on.

After adding the power supply you get something like this

![Amp2 on Raspberry](/assets/media/in-chassis.jpg){:width="60%"}

The bottom of the picture is also the bottom of the speaker. Hence if you close the chassis, the big speaker will be on the right.

In the middle you can see the power plug and the extended port glued to the back. From the back you can then access one USB port and the SD-card of the raspi.

![Amp2 on Raspberry](/assets/media/back-connectors.jpg){:width="60%"}

If you connect the buttons and soldered adapter from step 6 you get the following

![Amp2 on Raspberry](/assets/media/all-together.jpg){:width="60%"}

Hardware should be done now and we close up everything. If you have, you can glue a bit of padding to the cables and things inside. It is a speaker, so everything will be vibrating.

Now you have a Wifi enabled speaker build with freely available parts, maintainable and good looking.
Next step is to make it smart, which is basically choosing the right software.
[See next -> Open Source Wifi Speaker based on IKEA Symfonisk – Part II: Software / OS](/diy/2020/02/11/open-source-wifi-speaker-based-on-ikea-symfonisk-part-ii-software-os)
