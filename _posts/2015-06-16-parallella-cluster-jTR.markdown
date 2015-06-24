---
layout: post
title:  "Running JtR on our Parallella Beowulf Cluster"
date:   2015-06-15 11:42:13
tags: [parallella, cluster, epiphany]
---
In the previous post, we created a Beowulf cluster using *N* Parallella boards.
If you are using my images, the files you need to run the John the Ripper 
application should already be on them. If you don't (or if they aren't), 
follow this tutorial to set up the files on your cluster and get going. I'll
also walk you through how the code works. 

I wrote this demo for my parallel computing class, and presented it at the 
"Budget Beowulfs" special session at SIGCSE 2015. If you use these files, I 
ask that you please cite the following:

Adams J, Caswell J, Matthews SJ, Peck C, Shoop E, and Toth D. "Budget Beowulfs: 
A Show-case of Inexpensive Clusters for Teaching PDC". In Proceedings of the 
46th ACM technical symposium on Computer science education (SIGCSE’15). Kansas 
City, MO. March 6-8, 2015.

##Program Overview
Recall that one of the most common parallel patterns is the Master-Worker 
pattern, where a master process (or node) is responsible for creating a delegating 
a set of tasks to a series of worker processes. All *N* nodes then complete 
the work and report the final results back to the master. 

In the case of this demo, the "tasks" that need to be completed are the 
cracking of the passwords. Essentially, the program works as follows:

* The master node accepts a password file and a number of worker nodes (*N-1*).
* The password file is split into *N* parts. The master node and all worker nodes 
work to crack the passwords in parallel, writing the results to a local file.
* Once the work is complete, the master node collects and combines all the 
files. 

We have written a series of scripts to enable this entire process.

Are you ready? Let’s get started!

##Special Considerations
Some of you who are already familiar with parallel programming will likely be 
frowning at all the executable transfers and data transfers via SSH. After all, 
this step is often unnecessary when writing MPI applications on HPC clusters (save 
for the initial transfer to the cluster, and the offloading of the data from 
the cluster to a local machine). So why do we need it here?

The answer lies in the way we've set up our Beowulf clusters. Most HPC systems 
have a *common file system* that is shared by all the nodes in the cluster, 
and can be written to concurrently via the the network. In our case, we have 
four distinct Parallellas and thus four distinct file systems. All the SSHing 
that you see here is necessary to overcome this hurdle! In other HPC application 
that you may write in the future, it is all unnecessary.
 
##What you will need
This tutorial assumes that you have 2 or more Parallella Desktop edition 
computers. and have followed my instructons for [initial set up][setup], and 
[cluster setup][cluster]. If you are not using my image, make sure you also 
set up MPICH2 on your parallella master node prior to completing the cluster
set up tutorial. If you are not using my images, I strongly encourage you 
check out Simon Cox's excellent [Raspberry Pi cluster tutorial][rpi], which can be 
adapted for these purposes.

My image should have the files to run this demo. In case it does not (or if 
you are using different images), download the files in [my repository][repo] 
onto your master node and ensure that you have the following organization:
{% highlight bash %}
mpi_testing/
    |--- machinefile
    |----john_e_demo/
	 |-----	clean-up.sh
	 |-----	demo-e.sh
	 |----- get_files.sh
	 |----- split.py
	 |-----	transfer-files.sh

parallella-examples/
    |---john/
	 |-----	mpi/
		 |-----	john-mpi.c
		 |-----	transfer.sh
{% endhighlight %}

The `mpi_testing` directory should be a directory that exists on every Parallella.
This is incredibly important, as this is the main place where password files 
will be transferred to and cracked passwords are received. 


##Setting up the worker nodes with John MPI
Currently, our master node is the only one with the john-mpi.c file. Let's 
compile the code into an executable, and transfer the file to each node.
To do so, type in the following:
{% highlight bash %}
cd ~/parallella-examples/john/mpi
mpicc -o john_mpi john_mpi.c
{% endhighlight %}
Next, update the `transfer.sh` file so it contains the IP addresses of the 
worker nodes. Once you are done with that, type in:
{% highlight bash %}
./transfer.sh
{% endhighlight %}

Now, if you SSH into each of the individual worker nodes and look at their respective 
`mpi_testing/` directories, you should see a `john_mpi` there! Our master node 
even has a `john_mpi` executable in its `mpi_testing` directory.

###Running the Demo
Now we have everything we need to run the demo. Do the following:
{% highlight bash %}
cd ~/mpi_testing/john_e_demo/
./demo_e.sh mypass.txt
{% endhighlight %}
where `mypass.txt` contains the passwords you want to crack.

After some processing, you will see a file that shows up in the main directory:
`mypasscracked.txt`. This will contain the list of all the cracked passwords.

##Explanation of the code
Sadly, this part is going to have to wait for a while. One of my students is 
doing a project using JtR, and is using part of the code. As soon as that 
project is wrapped up, I will post the MPI file and go through the code in 
detail. 


[rpi]: http://www.southampton.ac.uk/~sjc/raspberrypi/ 
[cluster]: http://suzannejmatthews.github.io/2015/06/15/parallella-cluster/ 
[setup]: http://suzannejmatthews.github.io/2015/05/29/setting-up-your-parallella/
[repo]: https://github.com/suzannejmatthews/ejohn-beowulf-mpi-demo 
