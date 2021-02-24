---
layout: post
title: Add google traffic overlay to OSMAnd
date: 2021-02-23 00:00:00 +02:00
categories: SoWhatIsTheSolution
tags: [OSMAnd, Google]
fullview: true
description: Proper way to add a google traffic overlay to OSMAnd
author:
  name: Christian
  email: webmaster@christiansaga.de
published: true
---
OSMAnd is great with detailed maps, however when traveling I often miss traffic information.
Luckily you can add an online overlay, that will show the google traffic data.

I used guides like these [here](https://sites.google.com/site/tweakradje/android/osmand-navigation) or [here](https://freydanck.de/tag/live-verkehrs-daten-in-osmand/), however all I could find would add the traffic information PLUS a whole google map, street names or anything else. I wanted to just have the traffic information.

### So what is the solution

1. First of all to use online overlays, you need to activate the plugin "Online maps" in OSMAnd

2. After activating the plugin you can add in "Configure map" -> "Map source..." a new map:

![New map settings](/assets/media/OSMAnd.png){:width="60%"}

The URL string: `https://mt1.google.com/vt/lyrs=h,traffic&x={1}&y={2}&z={0}&apistyle=s.t%3A0|s.e%3Al|p.v%3Aoff`

The important part to remove labels and maps in here is the URL with the following config:
- lyrs=h,traffic - selects the street layer with traffic information. If you use s or m in the beginning you will get the satellite or maps view with the traffic information
- x, y and z - are the coordinates and zoom level filled by OSMAnd
- `apistyle=s.t%3A0|s.e%3Al|p.v%3Aoff`  - selects all feature styles (s.t), labels (s.e) and turns them off (p.v). Otherwise you will have street and location names in the overlay.

Especially the apistyle part was hard to find. Finally I found an explanation on [stackoverflow](https://stackoverflow.com/questions/29692737/customizing-google-map-tile-server-url) for the different possible settings.

Saving this map, then you can configure your map and activate an overlay map. If you need to change the traffic map settings you find them in the local maps for editing.
Be advised that this is an online function and requires a working internet connection and is also not included in route finding.
