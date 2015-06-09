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
