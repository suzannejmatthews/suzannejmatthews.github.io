---
layout: post
title:  "Raspberry Pi Cluster headless setup with VNC"
date:   2022-03-03 13:34:32
tags: [raspberry pi, raspberry pi cluster, rpi3, setup]
---

Yesterday, I was in Providence RI presenting a [workshop with the CSinParallel 
folks][workshop] at [SIGCSE'22][sigcse]. As part of the workshop, we demo'd the 
*self organizing clusters* that my students use at West Point (shown below):

![wpsoc](https://www.learnpdc.org/archive/WP-cluster.jpg  "West Point Self Organizing Cluster")

Before I continue, I should mention that these clusters are truly a reflection 
of the collaborative effort that is CSinParallel. Specifically:

* The [case design][thingi] was developed at West Point by myself and Frank Blackmon
* The self organizing cluster concept an initial image was developed at St. Olaf by Dick Brown and his students
* The [image][socimage] that the cluster uses was perfected at Macalester by Libby Shoop
* The patternlets concept that we use to teach concepts was developed at Calvin University by Joel Adams

After our successful workshop yesterday, a lot of people asked me for 
information on how to replicate our design at their own universities. I put 
together this short guide so faculty can replicate the West Point cluster 
(if they so choose) at their own universities.

## How we use the clusters
As I mentioned in my last post, the classroom I use for my course is currently 
equipped with a number of Windows desktops running Windows 10. I share this 
classroom space with a number of other professors, all who are teaching 
different courses, one after the other. Therefore, a dedicated Pi lab really 
isn't an option, and we don't want students to be tempted to 
disconnect/reconnect workstations peripherals as it can create problems for 
classes that take place after mine. 

We have extra powerstrips in the classroom for students to plug in their own 
devices, but a classroom of Raspberry Pi clusters can lead to a lot of power 
strips being utlized. The design that is discussed here is a "headless" cluster 
setup that allows two 3-node clusters to share a single 6-outlet power strip, 
in contrast to the 8 outlets that two clusters would normally require. There is 
an assumption that the workstation/laptop that the student is using has a 
USB C port, which we use to power the head node. 

### Raspberry Pi 3s
The other thing I should note is that our clusters use 
Raspberry Pi 3s, not the newer Raspberry Pi 4s. Obviously, it would be 
great to have Raspberry Pi 4s; however I (like many of you) have
had a really hard time getting ahold of the Pi 4 boards, thanks to the chip 
shortage. The Raspberry Pi 3s are easier to find, and you may have some 
lying around already. 

The classroom assessments that I've done show that students learn parallel 
computing concepts just fine from older Raspberry Pis. My one word of 
caution is not to use the Raspberry Pi 2s. Those *can* work, but 
my experiments suggests that the network connectivity on those boards are 
not as good as the Pi 3, leading to the head node getting randomly 
disconnected. I strongly recommend against using the even older boards, 
since I do not believe that the latest OS version for Raspberry Pi supports 
them. 


### Why 3 nodes instead of 2, 4 or more?

We picked three nodes for our clusters because we wanted to illustrate concepts 
on more than just two nodes. However, three nodes were less expensive than 
four. You can certainly create largers cluster with our image  -- the 
true limiting factor is the number of ports on the switch.

## Parts List to replicate one of our clusters
To replicate our cluster you will need:
* 3 Raspberry Pis ($35 each) - $105.00
* 3 microSD cards ($8 each) - $24.00
* 2 Raspberry Pi power supplies ($10 each) - $20.00
* 1 Gigabit 5-port Ethernet Switch ($12) - $12.00
* 4 1-ft Ethernet Cables ($2 each)  - $8.00
* 2 Ethernet to USB Adaptors ($10 each) - $20.00 
* A USB-C to microUSB (Raspberry Pi 3) charging cable OR
* A USB-C to USB-C charging cable (Raspberry Pi 4) - $10.00
* 4 3-D printed [baseboards][thingi] - high-density print
* 12 3-D printed [single connectors][thingi] - low-density print
* 3 velcro dots to connect Pis to switch - $0.20
* 4 little rubber feet (optional) - $0.50
* 12 #0/#0-80 machine screws - $0.50
TOTAL ESTIMATED COST: $200.20

I have no real estimate how much it costs to 3-D print the baseboards and 
single connectors, since my department has 3-D printers that I can use 
free of charge. However, I think it is generally pretty cheap. I have 
earlier posts that talk about commercial options for assembling clusters. 

Please note that this setup substitutes one of the Raspberry Pi power supplies 
with a USB-C charging cable. If your host computer does not have a a USB-C 
port, you should buy another power supply in lieu of the USB-C charging cable. 

## Step 1: Assemble the cluster

I pre-assembled my clusters prior to giving them to students. However, your 
students may have fun putting together the clusters themselves. If you 3-D 
print the case parts, it is recommended that you pre-thread the screw holes 
in the case to make it easier to attach the board. Again, another alternative 
is to just use acrylic baseboards and metal standoffs.[GeekPi][geekpi] has a 
commercial version for about $20.00 on Amazon, if you want to save yourself 
the trouble to 3-D print.

Essentially, the assembly instructions are as follows:
* Attach the Pis to the baseboards using screws, and then use the standoffs 
  to stack them on top of each other. At the end, you should have three
  Raspberry Pis stacked on top of each other, with a baseboard at the top, 
  and another at the bottom.
 
* Using the velcro dots, position the switch underneath the bottom of the 
  cluster. To get things to line up well, I usually put the dots on the 
  switch (velcro stuck together, sticky part up), and then just stick the 
  cluster to the switch. You have a little time to adjust the positioning 
  when you first stick it -- it is relatively easy to readjust while the 
  glue is still refresh. However, once the glue dries, it's a pretty 
  solid bond. 

* Next, use three of Ethernet cables to connect the ethernet ports on the 
  Raspberr Pis to the switch. 

* Take the last Ethernet cable, and connect the Ethernet adapters to either 
  end. Set aside for now.

## Step 2: Flash the Images
The next step is to flash the microSD cards. [Here is a link to the image][socimage].

* If you are having trouble downloading the image, ensure that your browser 
  isn't blocking downloads from http addresses. If it is, just right click on 
  the image link and click save. You should be prompted on whether or not 
  you really want to download. Say yes/keep to actually download it.

* Unzip the image from the downloaded zip file. When decompressed, the 
  image should be about 6GB in size.

* Use an application like Balena Etcher to flash the microSD cards.

* Once flashed, insert the microSD cards into each Pi

## Step 3: Final Setup
To save time and use the clusters in the context of a 2-hour lab period, I 
usually give my students the clusters with steps 1 and 2 above already complete. 
You could easily spend a 2-hour lab period assembling the clusters (some 
students may find this very enjoyable). 

Once steps 1 and 2 are done, it is time to boot up the cluster.

* Connect the power strip to an wall outlet and turn it on.

* Plug the switch into one of the outlets on the power strip

* Next, plug in the bottom two nodes into the power strip

* Connect the USB-C cable to your laptop/workstation, and put the other end 
  to the top node of the Raspberry Pi cluster. This will be our head node.

* Lastly place one end of the Ethernet adapter-ethernet cable-Ethernet 
  adapter daisy-chained cable into the top (head) node's USB port. Place 
  the other end into your laptop/workstation. 


## Step 4: Connect to the Pi (from Windows 10)
For the next set of steps, we will be following [the instruction][tut] on 
the [Raspberry Pi Cluster C MPI tutorial][tut]. 

Before we continue I want to briefly discuss how connections work in this 
cluster.

* The user/student connects to the Raspberry Pi head node via eth1 
  (this is the usb adapter to ethernet to usb adapter cabling system)
* The cluster uses eth0 to communicate with itself. 

The [image][socimage] that we provide has us connect to the cluster using 
the default Raspberry Pi username and password using SSH or VNC viewer. 

1. Ensure that the Pi is discoverable by attempting to ping it. Typing
   `ping 172.27.0.254` should result in a series of replies. 

2. Open up Windows PowerShell by clicking the Windows icon and typing in "PowerShell"

3. Confirm you can SSH into the Raspberry Pi by typing:
   `ssh pi@172.27.0.254` in Powershell. This should land you on the 
   command line of the Pi. If you are satisfied with your students 
   connecting to the Pi via command line you can stop here.

4. Download and Install the [VNC Viewer client][https://www.realvnc.com/en/connect/download/viewer/]
  
5. Open up VNC Viewer by clicking the Windows icon and typing "VNC Viewer"

6. In the top bar, type the hostname of the Raspberry Pi: `172.27.0.254`. 

7. Enter the username (`pi`) and the password (default is `raspberry`) 
   and click "OK". 


After that, your students should be all set to start using the [Raspberry Pi 
MPI tutorial][tut] that we shared during our workshop. You can get 
additional tutorials on the [learnPDC][learnpdc] website.

Have fun!

[sigcse]: https://sigcse2022.sigcse.org/
[workshop]: https://csinparallel.org/csinparallel/workshops/SIGCSE22_rpi_omp_workshop.html
[socimage]: http://selkie.macalester.edu/rpi_images/shrunk_csip_mpi_010622.img.zip
[thingi]: https://www.thingiverse.com/thing:892959
[geekpi]: https://www.amazon.com/GeeekPi-Cluster-Raspberry-Heatsink-Stackable/dp/B07MW24S61?ref_=ast_sto_dp&th=1
[tut]: https://www.learnpdc.org/RaspberryPi-mpi/index.html
[learnpdc]: https://www.learnpdc.org/
