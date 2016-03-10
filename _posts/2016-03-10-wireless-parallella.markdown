---
layout: post
title:  "Get Wireless on the Parallella"
date:   2016-03-10 2:42:23
tags: [setup, wireless]
---
I haven't posted here in a while, mainly since I've been very busy with
other things. Recently, I downloaded the updated image from Adapteva and 
started experimenting with it. Needless to say, the [image][image] that 
Parallella is now distributing is a LOT better than the one my students and 
I used last year during the course. For one thing, the new version of the 
kernel has support for wireless. Getting connecting to wireless is a cinch
using the new image.

### You will neeed: 
* A wireless USB adapter
* A powered USB hub
* USB female to microUSB cable

### Connecting to Wireless
While the machine is off, connect the USB adapter into the powered USB hub. 
Start up the machine, and open a terminal. Type in the following:
{% highlight bash %}
sudo nm-applet
{% endhighlight %}
This will start up the network manager client, which will appear as an icon 
on the lower right-hand portion of your screen. Click on that, and select 
your wireless network. If a password is required, it will try to connect 
and fail. At this point, close the network manager client and type in the 
following on the terminal:
{% highlight bash %}
sudo nm-connection-editor
{% endhighlight %}
A window will pop up, that includes your desired network. Highlight the network, 
and click the "Edit" button. You should now be able to enter a password for 
the associated SSID under the "WiFi Security" tab. Once you enter the password, 
click "Save". Start up network manger again (this time in the background):
{% highlight bash %}
sudo nm-applet &
{% endhighlight %}

You should now be able to use wireless!

[image]:     http://downloads.parallella.org/ubuntu/dists/trusty/image/ubuntu-14.04-hdmi-z7010-20140611.img.gz
[website]:   http://suzannejmatthews.com/

