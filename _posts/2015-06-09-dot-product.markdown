---
layout: post
title:  "The Dot Product Program"
date:   2015-06-09 14:50:33
tags: [parallella, epiphany]
---
Now that we have covered a basic Epiphany program, we will go on to a more 
complex example that executes in Parallel. This program can be found in the 
`epiphany-examples/apps/dotproduct/` directory on your Parallella board. 
Before we start discussing the code, let's discuss the problem.
 
##Calculating Dot Product in Parallel
The dot product between two arrays is the sum of the products. Consider the 
arrays *A=*[1,2,3] and *B=*[4,5,6]. The dot product of these two arrays is 1x4 + 2x5 +
3x6 = 4+10+18 = 32. A C implementation of this example follows:
{% highlight c %}
int main(){
  int a[3] = {1,2,3};
  int b[3] = {4,5,6};
  int i, sop=0;
  for (i = 0; i < 3; i++){
    sop+=a[i]*b[i];
  }
  printf("sop is: %d\n", sop);
  return 0;
}
{% endhighlight %}

Notice that our two arrays have the same length. To parallelize the dot product 
of two arrays over *n* elements and *c* cores we would do the following:

* Assign *n/c* elements of each array to each core. 
* Each core will then calculate a local sum of products using the the *n/c* 
elements assigned to it, and send it to the host.
* the host will sum all the local sums together to yield a final sum of products.

###An example
Consider the arrays *A =* [1,4,5,3] and *B =* [8,4,2,7]. We wish to calculate 
the dotproduct of *A* and *B* over two cores (*c=2*). Each array has *n=4* 
elements. 

To parallelize this program on Epiphany, we do the following on the host:

* The host initializes the device, and creates workgroup containing two cores.
* The host assigns the first two elements of *A* and *B* to core 1. Thus, the 
arrays [1,4] and [8,4] exist on core 1's local memory.
* The host assigns the last two elements of *A* and *B* to core 2. Thus, the 
arrays [5,3] and [2,7] exist on core 2's local memory.
* The host executes the device program on each core
* When the device program finishes executing, the host goes through each 
core and gets the local sums, and adds them together. This final sum is 
outputted to the user.

The device for its part does the following on each core:

* Each core steps through the portion of *A* and *B* assigned to it, and computes 
the sum of the products. So, core 1 computes *1x8 + 4x4 = 8+16 = 24*. Core 2 
computes *5x2 + 3x7 = 10+21 = 31*. 
* Once each core finishes computing the section of the original arrays assigned 
to it, it returns its local sum, and exits.

Make sure that this process is clear to you before we continue to discuss the 
dot product program. This is a classic case of task-based parallelism, and 
as you learn parallel computing, you will encounter many other programs that 
follow this pattern.

##The `dotproduct` example
The `dotproduct` example can be found the `epiphany-examples/apps/dotproduct/src` 
folder. Our device program is stored in the file `e_task.c` and the host 
program is stored in file `main.c`. We also have a header file called `common.h` 
which will contain the global `N` and `CORES` values that will be used by both 
the host and device programs. 

###The device program: `e_task.c`

Once again, for simplicity, we will start with the device program. 
{% highlight c linenos%}
#include <stdio.h>
#include <stdlib.h>
#include "e-lib.h"
#include "common.h"

int main(void)
{
  unsigned *a, *b, *c, *d;
  int i;
  
  a    = (unsigned *) 0x2000;//Address of a matrix 
  b    = (unsigned *) 0x4000;//Address of b matrix
  c    = (unsigned *) 0x6000;//Result
  d    = (unsigned *) 0x7000;//Done flag
  
  //Clear Sum
  (*(c))=0;

  //Sum of product calculation
  for (i=0; i<N/CORES; i++){
    (*(c)) += a[i] * b[i];
  }

  //Raising "done" flag
  (*(d)) = 0x00000001;

  //Put core in idle state
  __asm__ __volatile__("idle");
}
{% endhighlight %}

Like all device programs, we include the `e-lib.h` header file. We also include 
the `common.h` header file, which contains the definitions of `N` and `CORES`. 

On line 8, we declare 4 `unsigned` integer pointers. The first three, `a`, `b`, 
and `c`, we set to the addresses of three separate memory banks located on each 
Epiphany core. Recall that each e-core has 4 8KB memory banks, located at 
addresses `0x0000`, `0x2000`, `0x4000` and `0x6000`. The pointer `d` is set to 
address `0x7000`, which seems arbitrary, but well within the last memory bank.
Recall that `c` (unlike `a` and `b`) is not an array, but a static value.

On line 17 we initialize `c` to be 0. Lines 20-22 are nearly identical to our 
serial program, with `c` holding the result of the local sum of products between 
`a` and `b`. As far as I can tell, there is no significance to the parentheses 
around `*(c)`. I think it can equally be written as `(*(c))` or `*c`. Essentially,
since we want to update the *value* and not the *address* of c, we are using 
deferencing in this context.

Once we are done calculating our sum, we raise the done flag, `d`, by setting 
it equal to `0x1`. This is the cue to the host program that it can sum up all 
the values and output it to the user. At this point, we also place the core in
an idle state.

###The host program: `main.c`. 
Now let's move on to the host program:  


{% highlight c linenos%}
#include <stdlib.h>
#include <stdio.h>
#include <e-hal.h>
#include "common.h"

int main(int argc, char *argv[]){
  e_platform_t platform;
  e_epiphany_t dev;

  int a[N], b[N], c[CORES];
  int done[CORES],all_done;
  int sop;
  int i,j;
  unsigned clr;
  clr = (unsigned)0x00000000;

  //Calculation being done
  printf("Calculating sum of products of two integer unit vectors of length %d using %d Ccores.\n",N,CORES);
  printf("........\n");

  //Initalize Epiphany device
  e_init(NULL);                      
  e_reset_system();                                      //reset Epiphany
  e_get_platform_info(&platform);                          
  e_open(&dev, 0, 0, platform.rows, platform.cols); //open all cores

  //Initialize a/b input vectors on host side  
  for (i=0; i<N; i++){
    a[i] = 1;
    b[i] = 1;	  
  }
    
  //1. Copy data (N/CORE points) from host to Epiphany local memory
  //2. Clear the "done" flag for every core
  for (i=0; i<platform.rows; i++){
    for (j=0; j<platform.cols;j++){
      e_write(&dev, i, j, 0x2000, &a, (N/CORES)*sizeof(int));
      e_write(&dev, i, j, 0x4000, &b, (N/CORES)*sizeof(int));
      e_write(&dev, i, j, 0x7000, &clr, sizeof(clr));

    }
  }
  //Load program to cores and run
  e_load_group("e_task.srec", &dev, 0, 0, platform.rows, platform.cols, E_TRUE);
  
  //Check if all cores are done
  while(1){    
    all_done=0;
    for (i=0; i<platform.rows; i++){
      for (j=0; j<platform.cols;j++){
	e_read(&dev, i, j, 0x7000, &done[i*platform.cols+j], sizeof(int));
	all_done+=done[i*platform.cols+j];
      }
    }
    if(all_done==16){
      break;
    }
  }

  //Copy all Epiphany results to host memory space
  for (i=0; i<platform.rows; i++){
      for (j=0; j<platform.cols;j++){
	e_read(&dev, i, j, 0x6000, &c[i*platform.cols+j], sizeof(int));
      }
  }

  //Calculates final sum-of-product using Epiphany results as inputs
  sop=0;
  for (i=0; i<CORES; i++){
    sop += c[i];
  }

  //Print out result
  printf("Sum of Product Is %d!\n",sop);
  fflush(stdout);
  //Close down Epiphany device
  e_close(&dev);
  e_finalize();

  if(sop==4096){
    return EXIT_SUCCESS;
  }
  else{
    return EXIT_FAILURE;
  }
}
{% endhighlight %}

This program has a few, subtle errors. Do you see them? For now, we will 
discuss the host program as if it did not have any errors. Be sure to try out 
the exercises at the end of this post and read the next post for an explanation 
of the errors. 

I must point out that given the *assumptions* of this program (that operations 
are being performed solely on unit vectors) the errors are not very major. 
However, if you try to change the unit vectors to some other values, the 
program immediately breaks. We will discuss this in detail on how to improve 
the program in the next post. For now, let's just discuss its structure.

{% highlight c%}
#include <stdlib.h>
#include <stdio.h>
#include <e-hal.h>
#include "common.h"

int main(int argc, char *argv[]){
  e_platform_t platform;
  e_epiphany_t dev;

  int a[N], b[N], c[CORES];
  int done[CORES],all_done;
  int sop;
  int i,j;
  unsigned clr;
  clr = (unsigned)0x00000000;
{% endhighlight %}

The first fifteen lines of the program are very similar to the hello world 
program that we discussed previously. 

* Notice once again that we declare the mandatory library `e-hal.h`, which is 
required in all Epiphany host programs. We also include the header file 
`common.h`. 
* In our main function, we once again create our epiphany platform object 
(type `e_platform_t`) and our epiphany workgroup object (type `e_epiphany_t`). 
* In the next few lines, we declare our two local static arrays, `a` and `b`, 
whose lengths are set to `N`, from `common.h` (currently `N` is `4096`). 
* The static array `c` will contain the local sum of products collected from 
each of the `CORES` cores (`CORES` is currently set to `16` in `common.h`). 
* The integer `all_done` will allow us to to determine when all the e-cores 
finish with their work. 
* The variable `sop` holds the global sum of products, and represents the final 
value to be outputted to the user.
* `i` and `j` are just local variables that we will be using. The variable `clr` 
(which is not set to `0` for some strange reason), will be used for initialization 
purposes later.

Now, let look at the next few lines:
{% highlight c%}
  //Initalize Epiphany device
  e_init(NULL);                      
  e_reset_system(); //reset Epiphany
  e_get_platform_info(&platform);                          
  e_open(&dev, 0, 0, platform.rows, platform.cols); //open all cores

  //Initialize a/b input vectors on host side  
  for (i=0; i<N; i++){
    a[i] = 1;
    b[i] = 1;	  
  }
{% endhighlight %}

* The first three lines are mandatory for all Epiphany host programs, and 
respectively initializes the host library data structures, performs a full 
hardware reset of the Epiphany system, and gets information about the Epiphany 
chip.
* The line `e_open(&dev, 0, 0, platform.rows, platform.cols)` is a little
different that what we've seen in the Hello World program. Recall that the 
`e_open` command creates a workgroup of a particular size, starting at the 
(`row`, `col`) positions specified and going to the specified end coordinates.
In this particular case, we are instantiating the entire device as a single 
workgroup, so that all 16 cores will be utilized.
* The next four lines simply fills our `a` and `b` arrays with all `1`s. While 
this may seem counterintuitive at first, it will make sense as we go through 
the remainder of the example.

Moving on to the main body of the program:
 
{% highlight c%}
  //1. Copy data (N/CORE points) from host to Epiphany local memory
  //2. Clear the "done" flag for every core
  for (i=0; i<platform.rows; i++){
    for (j=0; j<platform.cols;j++){
      e_write(&dev, i, j, 0x2000, &a, (N/CORES)*sizeof(int));
      e_write(&dev, i, j, 0x4000, &b, (N/CORES)*sizeof(int));
      e_write(&dev, i, j, 0x7000, &clr, sizeof(clr));

    }
  }
  //Load program to cores and run
  e_load_group("e_task.srec", &dev, 0, 0, platform.rows, platform.cols, E_TRUE);
{% endhighlight %}
* For every core on the Epiphany device, the program writes the first `N/CORES` 
elements of the `a` and `b` arrays to the two memory banks located at positions
`0x2000` and `0x4000` respectively. These two lines are the source of many errors, 
**if** you try and change the values in our unit vectors `a` and `b`. Of course, 
in this case it doesn't matter, since each array only contains values of `1`. 
The `N/CORES` is also potentially problematic, but the program deftly avoids it 
by dealing with `N` and `CORES` values that are of a power of `2`. See the 
next post on how to fix this.
* The program does also initializes the 32-bit section of memory 
starting at location `0x7000` to 0. This corresponds to the done flag `d` in 
the device program, `e_task.c`. 
* The last thing that occurs in this section is that the device program 
specified by `e_task.srec` is copied over to the specified workgroup (`dev`), 
across all cores. The `E_TRUE` flag indicates that each core should execute 
its copy of `e_task.srec` immediately. Recall that every Epiphany host program 
will have a line that looks like this.

The next few lines has the host spinning until all cores on the device finish 
their respective computations:

{% highlight c%}
  //Check if all cores are done
  while(1){    
    all_done=0;
    for (i=0; i<platform.rows; i++){
      for (j=0; j<platform.cols;j++){
	e_read(&dev, i, j, 0x7000, &done[i*platform.cols+j], sizeof(int));
	all_done+=done[i*platform.cols+j];
      }
    }
    if(all_done==16){
      break;
    }
  }
 {% endhighlight %}
* The program continuously cycles through every core on the Epiphany device.
* For every core, it reads the 32-bit value stored at memory location `0x7000`
and places it in location `i*platforms.cols+j` of the `done` array. This value 
(which will either be `0` or `1`) is then added to the variable `all_done`. 
Honestly, we could have just read the value into a single integer variable, but 
I suppose this illustrates some math on how to read results into an array. 
* When all the cores raise their `done` flags, `all_done`, will be equal to the 
value of `CORES`. Another subtle error here is that the example hard-codes the 
values `16`. It should be `CORES`. The program does not break, because `CORES` 
is set to `16` in `common.h`.

Now that all computations are complete, we read the data, compute the global 
sum and output the result:
{% highlight c%}
  //Copy all Epiphany results to host memory space
  for (i=0; i<platform.rows; i++){
      for (j=0; j<platform.cols;j++){
	e_read(&dev, i, j, 0x6000, &c[i*platform.cols+j], sizeof(int));
      }
  }

  //Calculates final sum-of-product using Epiphany results as inputs
  sop=0;
  for (i=0; i<CORES; i++){
    sop += c[i];
  }

  //Print out result
  printf("Sum of Product Is %d!\n",sop);
  fflush(stdout);
 {% endhighlight %}
* Now that we know that each core has a local sum, we cycle through all the 
cores, and place the 32-bit local sum of products in the array `c`. For any core 
located at positions (`i`,`j`), we place it in location `i*platform.cols+j` in 
our array `c`. In this manner, all the data is placed in row-major order in 
the array. 
* The next four lines simply reads through our array `c` and calculates the 
global sum of product, storing the result in the variable `sop`.
* We then print out the value to the user. The use of `fflush` is curious here, 
since the `\n` when used in conjunction with `printf` on the previous line 
should have flushed the buffer.

The last few lines closes the device and does some error checking:

{% highlight c%}
  //Close down Epiphany device
  e_close(&dev);
  e_finalize();

  if(sop==4096){
    return EXIT_SUCCESS;
  }
  else{
    return EXIT_FAILURE;
  }
}
{% endhighlight %}

* All Epiphany host programs should have the lines `e_close()` and `e_finalize`, 
since these essentially closes the workgroup, and closes the channel with the 
Epiphany chip.
* The next few lines are **supposed** to do some error checking. The simplicity 
of using unit vectors is that the sum of products is necessarily `N`. Again, 
the example (incorrectly) hardcores the value `4096`. It would be more correct 
to change this to `N`. Once again, the program does not break because `N` is 
set to `4096` in the file `common.h`.

##Running the Code
Since we stepped through the `build.sh` and `run.sh` files in the Hello World 
example, we will not do so here. However, we will call to your attention some 
key differences in each file. 

In the build file, first notice that we are referring to `internal.ldf` instead 
of `fast.ldf`. Linker Descriptor Files (LDF)s help you choose the memory layout 
of your Epiphany application.

* If you are planning on storing everything external to the Epiphany chip, use 
`legacy.ldf`. 
* If you plan to have some memory outside (like in the case of our Hello World 
application, where we used some SDRAM memory, use `fast.ldf`.
* If all the memory (like in this application) is internal to the epiphany chip, 
use `internal.ldf`. 

##In Class Exercise
On your own, try and fix the Dot Product program, so that you can calculate the 
sum of products of arrays of size N where each array contains the values from 
0..N-1. Hint: you only need to update `common.h` and `main.c`!
