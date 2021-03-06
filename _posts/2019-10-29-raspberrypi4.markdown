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
learn more about it here: [http://www.diveintosystems.cs.swarthmore.edu][dis]

Of course, the other bit of news is that the Raspberry Pi 4 was released 
earlier this summer, and it not only boasts gigabit Ethernet, but 
configurations that support up to 4 GB of memory. I managed to get my hands 
on one of them, and I have to say, I've been having a LOT more difficulty 
with this version than previous Raspberry Pis. 
 
## Getting the monitor working (HDMI out of range error)

My first challenge was getting it to work with my older HDMI portable 
monitor, which has worked great with previous Raspberry Pi models without any 
issue. However, I got an "HDMI out of range" error when I tried 
to boot with the Raspberry Pi 4, regardless of what image I chose to use.
Very frustrating, since one of the key selling points of the Raspberry Pi is 
its "work out of the box" capability. 

I was able to fix the issue. If you run into something similar, start by 
inserting your microSD card into your computer and navigating to the `/boot` 
partition. The file you are typically looking for is `/boot/config.txt`. 
Please note that this is completely different from the `/rootfs/boot` 
directory, which is located on a separate partition (it will likely mount as 
two separate drives on a Desktop computer). If you don't see a `config.txt` 
file, it is likely that you are looking in the wrong partition.

Next, modify the `/boot/config.txt` file to include the following lines:
```
hdmi_force_hotplug=1
hdmi_drive=2
hdmi_safe=1
```
Save the file, reinsert into your Raspberry Pi 4 and commence rebooting.
You should now be able to see the desktop, and continue Raspberry Pi 
configuration as per normal.

Note: in some distros (e.g. [Ubuntu IOT][ubuntu]), the file you want to 
edit is `/boot/usercfg.txt`, NOT `/boot/config.txt`. In these cases, the 
`config.txt` file will warn you not to edit it directly, and point you to the 
file that should be edited instead. 

## Heat Issues

The next thing I noticed was that the Raspberry Pi 4 runs HOT. This is not an 
OS issue; the temperature of a Raspberry Pi 3B+ running the new Raspbian 
Buster OS stays consistent at around 75 degrees when doing `sudo apt get update && 
apt-get upgrade`. In contrast, the Raspberry Pi 4 running Buster runs 
significantly hotter, with the SDRAM chip reaching temperatures of 116 degrees.

Thankfully, a new firmware update has been made available that will fixes 
many of the issues. After running `sudo apt-get update && apt-get upgrade`, run 
the following two commands:

```
sudo apt-install rpi-eeprom rpi-eeprom-images
sudo reboot
```

Tom's Hardware has a [nice article][thw] explaining the issue in detail 
and how the firmware updates fix it.

## Comparing different images for the Raspberry Pi 4

[Raspbian Buster][buster] is (at the time of writing) the newest Raspbian 
release from the Raspberry Pi foundation. It will likely work well for most 
applications. Note that this is a 32-bit OS, like previous versions of 
Raspbian. If you want a 64-bit OS, you will need to do a bit more searching.

I also tried [Ubuntu Server IOT][ubuntu] for the Raspberry Pi 4. This has 
64-bit support; However, the image is very bare bones, and does not include 
many of the basic networking tools by default. Therefore, you will need a 
working Ethernet connection to complete setup. However, the 64-bit core 
release is still in beta, and the USB ports don't seem to work for the 4GB 
version. Ubuntu recommends adding the following line in 
`/boot/firmware/usercfg.txt` for 4GB models:

```  
total_mem=3072
```

This limits the memory on the 4GB model to 3GB. Again, this should not be an 
issue for the 1GB and 2GB models. [Some channels][omg] are reporting that while 
Canonical plans to extend full desktop support to the Raspberry Pi 4, it 
may be a while yet; the immediate focus will be Ubuntu Core.



 
[buster]: https://www.raspberrypi.org/downloads/raspbian/
[ubuntu]: https://ubuntu.com/download/iot/raspberry-pi
[dis]: https://diveintosystems.cs.swarthmore.edu/
[thw]: https://www.tomshardware.com/features/raspberry-pi-4-firmware-cool-temps-network-boot
[omg]: https://www.omgubuntu.co.uk/2019/11/ubuntu-raspberry-pi-4-support
