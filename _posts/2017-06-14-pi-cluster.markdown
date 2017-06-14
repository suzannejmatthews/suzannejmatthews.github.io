---
layout: post
title:  "Raspberry Pi Cluster NFS"
date:   2017-06-14 11:42:13
tags: [raspberry pi, cluster, setup]
---
Well, it's been a while since I've posted! It's another summer and I am 
creating yet another cluster. One thing people may not know is back in 
the summer of 2014 as I anxiously waited for the release of the Parallella
I was originally going to use Raspberry Pis in my parallel computing course.
Of course the Parallella came out, and so I put my Pis back on the shelf and 
concentrated on the Parallella for the course. 

While my love affair with the Parallella was intense and short, I have been 
using the Raspberry Pi for quite a few research projects involving students, 
to great success. I may eventually go back to the Parallella, but it occurred 
to me that some of my [old Raspberry Pi materials][pdf3] could also use an 
update, which was originally written in August 2014.

Unlike what I did with the Parallella tutorial I posted a couple years back, 
I'm *NOT* going to transcribe the entire tutorial from the PDF. The instructions
large still work. A couple things I should note: 

* my webpage lists images for the [Raspberry Pi 2][image2] and the [Raspberry Pi 3][image] 
  You can use the Raspberry Pi 3 image to create a cluster of Raspberry Pi 3s.
* you really should log into your router (192.168.1.1) to get the IP addresses 
to each of the nodes in your cluster. 

## Setting up a NFS shared hard drive
One of the things that I always struggle with when I put together these clusters
is the creation of the Network File System (NFS). The instructions are always a 
*little* different between image to image. So, the instructions below are for a 
cluster of Raspberry Pi 3s, though it is very likely they will apply to 
Raspberry Pi 2s as well.
 
As a reminder, pre-requisites for a NFS setup include:

* a hard drive that can be read/written to on Linux
* a working Raspberry Pi cluster (see above linked tutorial)

### Step 1: Install requisite software on master node
For the Raspberry Pi 3, the `nfs-common` and `rpcbind` packages are installed
by default. This makes it a *lot* easier to set up NFS when you already have a
Raspberry Pi cluster. Thus, the only thing you  need to install is the 
`nfs-kernel-server` on the master node:

{% highlight bash %}
sudo apt-get install nfs-kernel-server
{% endhighlight %}

### Step 2: Mount the hard drive on the master node
After you attach your external HD to the powered USB hub, see where it is 
located by using the `fdisk` command:
{% highlight bash %}
sudo fdisk -l        
{% endhighlight %}
A huge amount of output will result. The external hard drive will usually be at 
the end of this output. Mine is located at `/dev/sda1`. 

Next, create a mount point and mount your hard drive. In the examples below,
our mount point is `/mnt/nfs` and the hard drive is located at `/dev/sda1`: 
{% highlight bash %}
sudo mkdir /mnt/nfs
sudo chown -R pi:pi /mnt/nfs 
sudo mount /dev/sda1 /mnt/nfs       
{% endhighlight %}

### Step 3: Export new filesystem
The `/etc/exports` file allows us to define which folders/files we wish to 
share with others, and who we wish to share them with. Determine the IPs of 
all the worker nodes on your network by visiting `192.168.1.1`. In this 
example, our master node has the IP `192.168.1.123` and our three worker nodes 
have the IPs `192.168.1.124`, `192.168.1.125` and `192.168.1.126` respectively.

Modify the `/etc/exports` file to include the following lines:
{% highlight bash %}
/mnt/nfs 192.168.1.124(rw,sync,no_subtree_check)
/mnt/nfs 192.168.1.125(rw,sync,no_subtree_check)
/mnt/nfs 192.168.1.126(rw,sync,no_subtree_check)
{% endhighlight %}

This indicates that each of our worker nodes have read/write access to the 
shared drive located at `/mnt/nfs`, and all writes to the hd on the master
will be commited immediately to the network.

Let's restart some of the key services, and make this file visible on our 
network:
{% highlight bash %}
sudo /etc/init.d/rpcbind restart
sudo /etc/init.d/nfs-kernal-server restart
sudo exportfs -r
{% endhighlight %}

Now, log in on one of your worker nodes (say, 192.168.1.124):
{% highlight bash %}
ssh 192.168.1.124
{% endhighlight %}

We should be able to create our mount point and mount the hard drive:
{% highlight bash %}
sudo mkdir /mnt/nfs
sudo chown -R pi:pi /mnt/nfs 
sudo mount 192.168.1.123:/mnt/nfs /mnt/nfs       
{% endhighlight %}

You can check if it mounted correctly by using the ls command:
{% highlight bash %}
ls /mnt/nfs
{% endhighlight %}

If doing the above on a worker node shows you the contents of your harddrive, 
celebrate! All that is needed to repeat this process on all worker nodes.

### Step 4: Automate the mounting
When you reboot the machines, the hard drive will no longer be mounted. You 
can attempt to automate this process by editing the `/etc/fstab` file. However, 
if you ran into issues like I did, keep in mind that you can always script the
process. 

The sequence of commands that must always be run on the master are:
{% highlight bash %}
sudo mount /dev/sda1 /mnt/nfs
sudo /etc/init.d/rpcbind restart
sudo /etc/init.d/nfs-kernel-server restart
{% endhighlight %}

The sequence of commands that must always be run on each worker are:
{% highlight bash %}
sudo /etc/init.d/nfs-common restart
sudo mount 192.168.1.123:/mnt/nfs /mnt/nfs
{% endhighlight %}


## Troubleshooting
There are a number of problems you may face during setup. I've compiled a list 
of the most common issues, and ways to rectify them. 

* Check and make sure that all the ip addresses in `/etc/exports` are valid. 
If not, fix with the appropriate IPs. 
* Be sure to run the `exportfs -r` command once you update the `/etc/exports` 
file!
* Ensure that you can ssh into the worker nodes from the master nodes 
without needing a password. If you are asked to provide a password, that 
means the host’s ssh keys were not added to the worker’s set of authorized 
keys. Reboot the worker node!
* Note that the `rpcbind` and `nfs-kernel-server` have to be restarted each 
time. They do not start up automatically on the Raspberry Pi! 
* If it’s taking too long to build images for the worker nodes, consider using 
a smaller SD card to create the worker node image (I recommend 8GB). This 
will yield 8GB worker node images, which can be formatted onto a larger SD 
card. You can then expand the size of the image to fit the SD card.
See the set up tutorial for more instructions.




[rpi]: http://www.southampton.ac.uk/~sjc/raspberrypi/ 
[image2]: http://www.suzannejmatthews.com/private/cluster_master.7z 
[image]: http://www.suzannejmatthews.com/private/pi3-master.7z 
[pdf3]: http://www.suzannejmatthews.com/private/RaspberryPi_cluster.pdf 
