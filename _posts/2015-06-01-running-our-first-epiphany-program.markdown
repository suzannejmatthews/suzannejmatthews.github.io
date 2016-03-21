---
layout: post
title:  "Running our first Epiphany Program"
date:   2015-06-01 09:17:53
tags: [parallella, epiphany]
---
Before we get into the nitty gritty of programming the Epiphany architecture, 
we are going to do a quick demo and benchmarking study to illustrate how to 
use it. For the purposes of this demo (and cluster demo that will be posted 
later), we will be concentrating on the Epiphany implementation of John the 
Ripper, which was contributed by Katja Malvoni, and is accessible under the 
`parallella-examples/john` subdirectory on your Parallella board, or through 
the [parallela-examples][paragit] Git repository. 

## What is John the Ripper?
John the Ripper (JtR, or John) is a popular password cracking tool that can be used to 
crack "weak" UNIX passwords. A weak UNIX password is made up of common words, 
people's first and/or last names, and common sequences like `1234`. Most 
passwords are kept safe by using encryption schemes like BlowFish that generate 
a hash. So, if a company is hacked and passwords are stolen, the passwords 
are usually of the following form:
{% highlight text %} 
user0:$2a$05$GDX0k9mentz8lrGB9vYMbOj31ft.tRXsJetNcRrRzuW4es5U9hGUa
user1:$2a$05$DjSZDk44.Pmqq5qJXx9Y4.x1kOwWUSxt5nyR4cWgO6GNdTywlUJBa
user2:$2a$05$UBw1ZBPQKNR6vugOW5Ax2O.gAqpqYHt8kL.HgiBLutQ2fVOVgaiKG
user3:$2a$05$h0XpSSvQektk2AHpoVho.eMGTizWjBoCw2VgXVIdaeAcg.UFEfS2y
user4:$2a$05$9dlM5U90sChXaTuLLWN1v.2M6rxwHoL4GdpEAxQXtSbPlA678g6S2
user5:$2a$05$8knQE/7/v61tDM4SbWMh3ut4zuGWdmqKlS5Mod6m0djaoA1Izh3Fy
{% endhighlight %} 
Each line in this file represents a separate username, followed by a colon, 
followed by a hashed password. As an added protection from Rainbow table 
attacks, these hashes are usually *salted*. A *salt* is a special sequence of 
random data that is placed ahead of a password before it is hashed. Since the 
attacker does not know what the salt is, it makes it harder for them to figure 
out what the real password is. The complexity of the salt refers to the set of 
possible values it can take on. Thus, a 5 bit salt takes on 32 possible values 
while a 12-bit salt takes 4,096 values. 

As you can imagine, bruteforcing such passwords can take a very long time. 
But John is very clever. It employs a *dictionary* style attack to quickly break 
passwords. In a dictionary attack, the attacker *has* a list of known, weak 
passwords. The sofware then hashes each of the know weak passwords and compares 
it to the list of passwords in the stolen hash file. If there is a match, then 
we know what the password is. 

Breaking the salt is the hardest part of all of this. If there are *n* separate 
salts for *n* separate passwords, the breaking of passwords can take a very 
long time. However, if the company uses a single salt or *k* salts where 
*k << n*, then John can simply "rip" through the list of passwords, hence its 
name. 
 
## How does the John Epiphany program work?
To understand how John works on Epiphany, you have to pay attention to the 
Epiphany architecture. We will get into that in greater detail in the next 
post. For now, it suffices to say that the Epiphany co-processor has 16 total
cores. This essentially enables 16 passwords to be cracked simultaneously, 
where each core independently works on cracking its own password. This means 
we should speedup of up to 16 on the Epiphany architecture!

So let's crack some passwords! For now, paste the above code into a file, 
`tocrack.txt` and place it in the `parallella-examples/john/run` directory. 
Note that there is another file, `password.lst` that is included that contains 
a list of weak passwords. 
Open up a terminal, and type the following:
{% highlight bash %} 
cd /home/linaro/parallella-examples/john/run
sudo -E LD_LIBRARY_PATH=$LD_LIBRARY_PATH ./john tocrack.txt --wordlist=password.lst -form=bcrypt-parallella
{% endhighlight %} 
Press `Enter`. While we wait for JtR to do its thing, let's analyze this command 
more closely, because there is a lot going on here:

* All Epiphany programs must be run with the `sudo` prefix. The `-E` flag 
allows us to specify the environment, which we set to reference the location 
of where the Epiphany libraries are installed on our system.
* The next part of the command, `./john tocrack.txt --wordlist=password.lst`, 
is the JtR specific portion. This is the same as if we were to run John on 
a regular system. The command tells John to crack the file `tocrack.txt`, using 
the plaintext passwords in `password.lst` as the reference.
* The last portion of the command, `-form=bcrypt-parallella` is a special 
command to Epiphany. This is telling the system that the encryption scheme 
that is being used is Blowfish. While the regular implementation of JtR 
supports multiple encryption types, including AFS, md5, and BSDI, the Epiphany
program currently only supports Blowfish.

If all goes well, you should see the following show up (your order may be different):
{% highlight bash %} 
Loaded 6 password hashes with 6 different salts (bcrypt [Blowfish 32/64 X2])
Press 'q' or Ctrl-C to abort, almost any other key for status
legend		(user4)
jamjam 		(user0)
miranda 	(user1)
vermont		(user2)
katrina		(user3)
cessna		(user5)
{% endhighlight %} 

## Benchmarking Results
I did some extensive benchmarking analysis earlier in the school year, but want to 
double check my numbers. I will post the results soon.

[paragit]:      https://github.com/parallella/parallella-examples
