---
layout: post
title:  "Raspberry Pi 4"
date:   2019-10-29 13:21:05
tags: [raspberry pi, rpi4, setup]
---

I know it's been a while since I've posted, but I'm still working on Raspberry 
Pi (and Raspberry Pi related projects). My latest project is _Dive into Systems_, 
a free on-line textbook for introductory computer systems concepts that I 
co-author with Swarthmore professors Tia Newhall and Kevin Webb. You can 
learn more about it here: http://www.diveintosystems.cs.swarthmore.edu

Of course, the other bit of news is that the Raspberry Pi 4 was released 
earlier this summer, and it not only boasts gigabit Ethernet, but 
configuration that support up to 4 GB of memory. I managed to get my hands 
on one of them, and I have to say, I've been having a LOT more difficulty 
with this version than previous Raspberry Pis. 
 
## Getting the monitor working (HDMI out of range error)

My first challenge was getting it to work with my old GeChic HDMI portable 
monitor, which has worked with previous Raspberry Pi models without any 
issue. However, I got an "HDMI out of range" error when I tried 
to boot with the Raspberry Pi 4, regardless of what image I chose to use. 
If you run into the same issues, you may have to edit the `config.txt` file 
in `\boot` (to access it, plug your microSD card into your PC. One of the 
partitions that will come up is the `boot` partition).  To get the monitor 
to work with Raspbian Buster, I had to uncomment the following lines:

----
hdmi_force_hotplug=1
hdmi_drive=2
hdmi_safe=1
----

And save the file. Rebooting finally let me see the desktop. I was able to 
continue configuration from there. Adding the above lines to the 
`/boot/usercfg.txt` file in [ubuntu][Ubuntu] also enabled the monitor to 
start working. 

## Comparing different images for the Raspberry Pi 4

[buster][Raspbian Buster] is (at the time of writing) the newest Raspbian 
release from the Raspberry Pi foundation. For most applications, you will 
likely find that it works well. However, like previous versions of Raspbian, 
the OS is 32-bit. If you need a 64-bit OS for whatever reason, you will 
need to keep looking. 

I also tried [ubuntu][Ubuntu Server IOT] for the Raspberry Pi 4. This has 
64-bit support; However, the image is meant to be bare bones, and is 
missing most of the networking tools by default.  As such, you will need an 
Ethernet connection to finish setup. 

[buster]: https://www.raspberrypi.org/downloads/raspbian/
[ubuntu]: https://ubuntu.com/download/iot/raspberry-pi

