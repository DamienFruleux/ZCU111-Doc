You can found a python exemple (with Pynq) [here](https://pynq.readthedocs.io/en/latest/pynq_libraries/dma.html), or a C exemple [here](https://lauri.xn--vsandi-pxa.com/hdl/zynq/xilinx-dma.html).

Pynq uses dynamic memory allocation, which means that it is not possible to use all available memory, let alone use it continuously.

Note that if it is possible to make continuous transfers in Python as in C, Pynq does not yet offer this method directly (to my knowledge). 


## Dynamic allocation, mmap & 32Go PSRAM

As I explained above, the pynq_notebook uses the Pynq libraries. Although it works perfectly, there are some limitations that can become annoying in some cases. 

In particular, I have not yet found a way to use PLRAM (but it seems that it is possible).

Moreover, by choosing PSRAM, the memory allocation is done dynamically (malloc function in C), but is limited to 128MB. In order to extend this limitation, it is necessary to compile the Kernel by modifying some options (more info [here](https://discuss.pynq.io/t/pynq-maximum-allocatable-memory-cma/1593), but I intend to add some documentation on this task later).

Finally, even if it is possible to allocate the desired amount of memory, it is not certain that the data are arranged continuously in memory, which could potentially cause some problems since the analog signal is [continuous]() over time: we could then see artifacts appear in the analog signal. 

In order to overcome these problems, it is possible to use the standard python libraries (or the C language in my case). 
It is then necessary to "open" the memory (function open /dev/mem) and "map" the memory address space which interests us (function mmap). 

Although it works perfectly and without any problem in the case of PLRAM because it is not used by the Linux Kernel, you have to be careful when using PSRAM. 

Indeed, by default the ZCU111 has a 4GB SODIMM. If we use the method described above, we run the risk of using memory addresses potentially used by the Linux Kernel. As I needed a lot of memory and not to use dynamic allocation (data continuity), I upgraded the SODIMM up to the maximum possible (32GB) and I made sure that it is well mapped in /dev/memory, but without the Linux kernel being able to exploit it like classic RAM. More simply, linux still sees 4GB of RAM, but can access the other 28GB of the PS in the same way as the 4GB of the PL.

Note however that it is possible to use the 32GB of RAM with the Linux Kernel and to allocate the necessary amount of memory dynamically. I cannot however certify the continuity of the data in memory. (Note that with a kernel modulator and the kmalloc function this seems possible). 
Finally, this "method" allows me to use a single code to use the PS and the PL, since only the memory address space differs. 



***

Moreover, C being much more powerful than python, it can be necessary in some cases. In particular it is necessary to make sure that the data is sent fast enough to the DMA, in order to be sure to have a continuous signal to send to the DAC.

***




### 1- Use PSRAM & Pynq (dyn alocation - limited to 128Mo)

### 2- Use PLRAM & Pynq (don't work for the moment, see later)

### 3- Use PLRAM & C/Python

### 4- Use PSRAM & C/Python (dyn alocation - don't work for the moment, see later)

### 5- Use PSRAM & C/Python

# Continuous transfert