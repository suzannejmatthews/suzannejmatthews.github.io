---
layout: post
title:  "Raspberry Pi Cluster - Image Creation with MPI"
date:   2018-07-18 10:40:48
tags: [raspberry pi, cluster, setup, image]
---
I have a few more posts in mind for this summer, but I wanted to create 
this one while the information was still fresh in my head. Imagine my 
dismay when I discovered that the [original Raspberry Pi cluster tutorial][rpi] 
created by Dr. Simon Cox at the University of South Hampton [no longer 
seems to exist][rpi]. This has always been to me the de-facto go-to tutorial 
for creating Raspberry Pi images. I've used these instructions multiple 
times to create the images that I post on my website for those who are 
interested in using ready-made Raspberry Pi images for classroom use. I even 
adapted my [own tutorial][pdf3] from these instructions back in 2014.

So where does that leave us, dear reader? What to do if we wish to create 
future images? To assist all of you in your own custom cluster projects, 
I'm reproducing the instructions for the original tutorial. 


## Getting things ready
Before you start, I encourage you to buy a couple of 8 GB microSD cards. In my 
experience, the full image (with MPI etc.) doesn't fit nicely into a 4 GB 
microSD card, so 8 GB is the next smallest size. You want to pick the smallest 
microSD cards you can, since it will save a lot of time in the image creation 
phase. You will also need a laptop with an SD card slot, or an SD card to USB 
reader.

1. I first use [SD Card Formatter][sdformat] first to get rid of anything that could be on the 
   microSD card to begin with. 
2. Next, download the latest copy of Raspbian. The one I've downloaded and used 
   for this tutorial is Raspbian stretch.
3. Use [Win32DiskImager][win32] or something similar to "burn" the downloaded image onto 
   your Raspberry Pi. 

Once you complete these three steps, you will be in good shape to start the 
process to install MPI.

## Creating the Master Node

Once you've booted up your Pi, first set up things in `raspi-config`

1. Enter `sudo raspi-config` in the terminal
2. If you are not in the UK, select option 4, `Localisation options` Use that 
   to set the keyboard layout, locale, time zone and WiFi Country. 
3. Next, select option 5, `Interfacing Options`. Navigate to `SSH` and choose 
   `Yes` and `Ok` then `Finish`.

This is a crucial step, because the WiFi country must be enabled in order for 
you to connect your Pi to WiFi. Lastly, the SSH server is not enabled on the 
Pi by default any longer, so you absolutely must enable it using option 5.


Once you've completed the above steps, connect your Pi to the Internet, and 
download [MPICH][mpich]. While I certainly encourage you to try the latest 
version of MPICH, I've run into problems with MPICH3 on earlier version of 
the Pi. Perhaps these issues have been have been resolved on the Raspberry Pi 
3B+, but I decided not to risk it. The instructions below show what you need 
to do to install MPICH2 on the Raspberry Pi, but you just need to change the 
`wget` location to change the instructions for the latest version of MPICH.

In the home directory, type in the following commands:
{% highlight bash %}
wget http://www.mpich.org/static/downloads/1.5rc2/mpich2-1.5rc2.tar.gz
tar -xzvf mpich2-1.5rc2.tar.gz
{% endhighlight %}

Next, let's create some folders that will help us with installation:
{% highlight bash %}
mkdir mpich2-install mpich2-build
{% endhighlight %}

Next, we are going to `cd` into the `mpich2-build` directory, and start the 
configure process. It is absolutely important that you do this in the `build` 
directory, because otherwise files will install in the `home` directory, which 
may mess with your permissions. 

In these next steps, I'm also choosing not to install fortran77, since all the 
MPI programming I do is in C. If you want to have fortran, I encourage you to 
install `gfortran` ahead of time and to ignore the `-disable-fc` and `-disable-f77` 
flags that I added to the configure command below:

{% highlight bash %}
cd mpich2-build
/home/pi/mpich1.5rc2/configure -disable-fc -disable-f77 -prefix=/home/pi/mpich2-install
sudo make
sudo make install
{% endhighlight %}

Next, we need to update the `PATH` variable in `.bashrc` so we will be 
able to access the MPI executables anywhere. From the home directory:

1. Open up `.bashrc`
2. Add to the end of the file the line: `PATH=$PATH:/home/pi/mpich2-install/bin`

Once you are done, test out `mpiexec` using the following command:

{% highlight bash %}
mpiexec -n 2 hostname
{% endhighlight %}

You should see `raspberrypi` show up two times. If you get this far, celebrate!
You have successfully created the master node.

## Creating the first worker image

Now, we will create a worker image that we will use to build all the worker 
nodes for our Pi cluster:

1. Using win32diskImager or similar, "read" the contents of the master node 
   and save it onto your desktop. Name it `master.img` or similar.

2. Next, insert a new 8 GB microSD card. Burn `master.img` onto the new microSD 
   card using the "write" command. 

3. Once you are done, network together the worker node and the master node, 
   by connecting them to a router. When rebooting your "cluster", ensure that 
   the router comes up first, followed by the Pis. I like to boot my router, 
   and then connect my Pis to power afterward. Alternatively, you can boot 
   everything up together, and connect the ethernet cables after the router 
   fully boots up.  

4. Let's double-check that MPI is still working as expected. Type in the following 
   commands:
   {% highlight bash %}
   mkdir mpi_test && cd mpi_test
   ifconfig
   {% endhighlight %}

5. Once obtain your ip address using `ifconfig`, let's actually add it to 
   a new file called `machinefile`. In the example that followed, we assume that 
   the ifconfig command returns `192.168.1.101`

6. {% highlight bash %}
   echo "192.168.1.101" > machinefile
   mpiexec -f machinefile -n 2 ~/mpich2-build/examples/cpi
   {% endhighlight %}
   You should get some output showing an approximation of pi. If you get here, 
   celebrate!

7. Next, we want to generate ssh keys. Go to the home directory and type in 
   the following commands:
   {% highlight bash %}
   cd
   ssh-keygen -t rsa -C 'pi@raspberrypi'
   {% endhighlight %}
   Keep pressing enter to accept the defaults. Do NOT enter a passphrase!

8. Login to your router (usually at `192.168.1.1`) and check the IP address 
   of your worker node. Suppose the worker's IP is `192.168.1.102`. Ensure 
   the node is reachable by using the ping command: `ping 192.168.1.102`

9. If the node is reachable, let's now try and ssh into it:
   `ssh 192.168.1.102`. You wil get prompted for the passphrase. Just CTRL-C
    it for now. 

10. Next, type in the following command:
    {% highlight bash %}
    cd
    cat .ssh/id_rsa.pub | ssh 192.168.1.102 'cat >> .ssh/authorized_keys'
    {% endhighlight %}
    
    This will place the public key into the set of authorized keys for the 
    worker node. 

11. Now, try and re-SSH into the worker: `ssh 192.168.1.102`. You should now 
    be able to access it without a password!

12. You should be now connected to the the worker node. Let's change its 
    hostname. Type `sudo nano /etc/hostname` to launch the nano editor. 
    Replace the hostname with something like `worker001`. Restart the machine
    to see the new changes:
    {% highlight bash %}
    sudo shutdown -r now
    {% endhighlight %}
    
    I would also recommend changing this in `/etc/hosts` next to the local 
    host IP.

13. Once the worker node comes back up, we can rerun the CPI example from 
    earlier: 
    {% highlight bash %}
    cd mpi_test
    echo "192.168.1.102" >> machinefile
    mpiexec -f machinefile ~/mpich2-build/examples/cpi               
    {% endhighlight %}
    You should now see the same output from before, but now with two different 
    hostnames! Great work!

## Burn and Churn

Our worker node now contains the authorized keys for our master node. Burn a 
copy of your worker node onto your desktop, using Win32DiskImager or something 
similar and the `read` command. Next, burn the image onto a new microSD card 
using the `write` command. 

All that remains to be done is:

1. Connect the new worker node to the cluster via the router.
2. Discover the new worker node's IP address using the router homepage.
3. SSH into the new worker node (you shouldn't need a password!)
4. Change the hostname in `/etc/hostname` and `/etc/hosts`.
5. Reboot the new worker node
6. Add new IP to `machinefile` and rerun the CPI example from above

You  now have a working Raspberry Pi cluster that runs MPI! Congratulations!

## Troubleshooting

Here are some tips if you run into trouble:

*Q: ssh-keygen is suddenly failing! What do I do??*
A: A big difference between Raspbian Stretch + later vs. prior releases of 
   Raspbian is that the SSH server is not turned on by default. Ensure that 
   you run `sudo raspi-config` and enable SSH through option 5. 

*Q: When I try to SSH into the worker node, I get a message saying, "connection 
    refused". What did I do wrong?*
A: Ensure the worker pi is plugged in. If it, check the ethernet port. Is it 
   receiving data? Check the router home page. Does it have an IP address? 
   Can you ping it? If any of the above fails, your worker node is not properly 
   connected to the router. Next, double check and ensure that you do have 
   the SSH server installed on the worker nodes as well. If you did the step 
   *AFTER* you burned the master image (instead of *before*, like specified in 
   the instructions, you may have to turn it on the workers as well. This is 
   the main reason why this command would fail.

[rpi]: http://www.southampton.ac.uk/~sjc/raspberrypi/ 
[sdformat]: https://www.sdcard.org/downloads/formatter_4/ 
[win32]: https://sourceforge.net/projects/win32diskimager/ 
[pdf3]: http://www.suzannejmatthews.com/private/RaspberryPi_cluster.pdf 
[mpich]: http://www.mpich.org/downloads/
