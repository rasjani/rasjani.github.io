---
layout:	post
title:	"Enforce 1080p resolution through VGA -> HDMI box"
date:	2015-07-23
tags: [linux, hardware]

---

  I bought a new TV few days ago and i knew i was going to be having some issues with my mediabox setup. Since I'm cheapskate and/or just haven't bothered to upgrade my living room computer from stone age vga only card to anything that would have HDMI output, i was forced to buy a [small vga to hdmi scaler](http://www.verkkokauppa.com/fi/product/20954/fjgqd/Fuj-tech-signaalinmuuntaja-VGA-audio-HDMI-musta) in order to even connect my kodi box to the new tv.

Since I didn't reboot the box before connecting to the tv set, everything was working fine and i didn't really think about what would happen when someone reboots my humble Ubuntu box that is running my Kodi setup. Few days later it happened =)

And yey, wonderful world of 1024x768 and hour or so hacking and trying out things. Since i haven't really been running and keeping myself up2date with all the changes since .. probably 10.10 Ubuntu release and X11, ofcourse i started my debugging from xorg.conf. Which i wasn't able to find. WTF?! Quick googling reveals that this file is not used anymore because everything should be detected automatically. Since my last (Sony) TV set did actually have VGA input, Xorg was able to detect everything just fine but now, connection to HDMI via this box was crapping out. After some searching in tah intarwebz and i came up with this:

Run following command:

```bash
cvt 1920 1080 60That command will output something like this:
```
command will output something like this:

```
# 1920x1080 59.96 Hz (CVT 2.07M9) hsync: 67.16 kHz; pclk: 173.00 MHz  
Modeline “1920x1080\_60.00” 173.00 1920 2048 2248 2576 1080 1083 1088 1120 -hsync +vsync
```

The interesting line in that is the row that starts with Modeline. Next, open/create a file /usr/share/X11/xorg.conf.d/10-monitor.conf and copy paste following into it (replace everything if you have something in there). Do note that this file can only be edited as root.

```
Section "Monitor"  
 Identifier "Monitor0"  
 <INSERT MODELINE HERE>  
EndSection  
Section "Screen"  
 Identifier "Screen0"  
 Device "<INSERT DEVICE HERE>"  
 Monitor "Monitor0"  
 DefaultDepth 24  
 SubSection "Display"  
 Depth 24  
 Modes "<INSERT MODENAME HERE>"  
 EndSubSection  
EndSection
```
The thing here is that you should now replace the whole row that says <INSERT MODELINE HERE> with the modeline from cvt's output. And <INSERT MODENAME HERE> should be replaced with the first string in quotes of the same modeline row.

And when you have done that, reboot and you should be able to get 1920x1080 resolution.

PS. If your monitor/tv panel does not support 1080p resolution, just change the cvt command's two first parameters with resolution that your tv supports.

