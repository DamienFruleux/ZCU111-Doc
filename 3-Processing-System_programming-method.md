# Introduction

There are different ways to proceed for the "data management". 

The easiest way is certainly to use a pynq notebook and the [dma class of the Pynq library](https://pynq.readthedocs.io/en/v2.6.1/pynq_libraries/dma.html), but this has some limitations.

That's why I developed a C code that does almost the same thing, but in a lower level, using the standard POSIX libraries ([example 1](https://lauri.xn--vsandi-pxa.com/hdl/zynq/xilinx-dma.html) and [example 2](https://www.hackster.io/whitney-knitter/introduction-to-using-axi-dma-in-embedded-linux-5264ec)). Finally, it is possible to do the same thing as in C but in Python (it may be easier for some people).

As there are 2 types of memories (PS and PL) that can be used in each case, I will detail the different possibilities. Be careful though, not all possibilities are possible, especially with Pynq.

# General operation of the Vivado design

todo : PS->DMA->DAC & ADC->DMA->PS

# Use Pynq libraries

As I explained, the pynq_notebook uses the Pynq libraries. Although it works perfectly, there are some limitations that can become annoying in some cases. 

## Program Operation

The explanations of [dma class of the Pynq library](https://pynq.readthedocs.io/en/v2.6.1/pynq_libraries/dma.html) are very well explained and illustrate very well the different steps required.

## Limits with PSRAM

### 1- Dynamic memory allocation

Pynq uses dynamic memory allocation (malloc function in C), which means that it asks the kernel to allocate a certain amount of memory. So there are limits on the amount of memory allocated, since the OS must keep a certain amount of memory: it is not possible to allocate all 4GB of RAM to store data !

### 2- Buffer size

By default, it is not possible to allocate more than 128MB of data using the "allocate" function provided by pynq : [This buffer is allocated inside the kernel space using xlnk driver. The maximum allocatable memory is defined at kernel build time using the CMA memory parameters. For Pynq-Z1 kernel, it is specified as 128MB] (https://pynq.readthedocs.io/en/v2.0/_modules/pynq/xlnk.html).

This documentation is from version 2.0 of pynq, it is possible that the size has increased, especially for the ZCU111, but in any case, the principle remains the same: there is a significant size limit.

In order to extend this limitation, it is necessary to compile the Kernel by modifying some options, in particular : 

- Device Drivers->Generic Driver Options->Size in Mega Bytes(1024)

- Device Drivers->Staging drivers (ON)->Xilinx APF Accelerator driver (ON)->Xilinx APF DMA engines support (ON) 

You will find more information [here](https://discuss.pynq.io/t/pynq-maximum-allocatable-memory-cma/1593), but I intend to add some documentation on this task later.

### 3- Data arrangement in memory

Finally, even if it is possible to allocate the desired amount of memory after some manipulations, it is not certain that the data are arranged continuously in memory, especially if there are large amounts of data, which could potentially cause some problems since **the analog signal is continuous over time** : we could then see artifacts appear in the analog signal, like holes.

### 4- Continuous transfert

Note that if it is possible to make continuous transfers in Python as in C, Pynq does not yet offer this method directly (to my knowledge).

I have previously how to perform [continuous transfers](https://github.com/DamienFruleux/ZCU111-Doc/blob/main/1-AXI_DMA.md#use-a-fifo), in order to get rid of the [DMA limit of 67 108 863 bytes](https://github.com/DamienFruleux/ZCU111-Doc/blob/main/1-AXI_DMA.md#maximum-size-of-a-transfert). However, due to the very high level operation of the pynq functions, it is almost certain that the time between the end of the execution of the 1st function *dma.sendchannel.transfer(input_buffer_a)* and the time between the beginning of the 2nd function *dma.sendchannel.transfer(input_buffer_b)* will be too long to allow a continuous transfer. I haven't tested this theory yet, maybe I'll do it sometime. 

## Limits with PLRAM

With Pynq, to use the PLRAM, the [mmio](https://pynq.readthedocs.io/en/v2.6.1/pynq_libraries/mmio.html) class must be used. In particular, to write data, you must use the function **mmio.write(address_offset, data)**.

The operating principle of the [dma](https://pynq.readthedocs.io/en/v2.6.1/pynq_libraries/dma.html) class requires that the **dma.sendchannel.transfer()** function takes an input_buffer as argument. 

These particularities have not yet allowed me to use PLRAM as memory to store data before sending them to the DACs (or after receiving them from the ADCs), with Pynq.

Finally, although it seems possible, the documentation does not seem to consider this possibility as interesting : [MMIO provides a simple but powerful way to access and control peripherals. For simple peripherals with a small number of memory accesses, or where performance is not critical, MMIO is usually sufficient for most developers. If performance is critical, or large amounts of data need to be transferred between PS and PL, using the Zynq HP interfaces with DMA IP and the PYNQ DMA class may be more appropriate](https://pynq.readthedocs.io/en/v2.6.1/pynq_libraries/mmio.html).

If you have any information, don't hesitate to share it with me.


# Use C/Python standard libraries

## Program Operation

In order to overcome these limitations, it is possible to use the standard POSIX libraries, in C or python language. 

I resume the explanations of [Lauri Vosandi](https://lauri.võsandi.com/hdl/zynq/xilinx-dma.html), which are in my opinion perfect: 

```
When it comes to writing C code I see alarming tendency of defaulting to vendor provided components: 
stand-alone binary compilers, Linux distributions, board support packages, wrappers while avoiding learning what actually happens in the hardware/software.

As described in my earlier article physical memory can be accessed in Linux via /dev/mem block device.
This makes it possible to access AXI Lite registers simply by reading/writing to a memory mapped range from /dev/mem.
To use DMA component minimally four steps have to be taken:

* Start the DMA channel (MM2S, S2MM or both) by writing 1 to control register

* Write start/destination addresses to corresponding registers

* To initiate the transfer(s) write transfer length(s) to corresponding register(s).

* Monitor status register for IOC_Irq flag.

In this case we are copying 32 bytes from physical address of 0x0E000000 to physical addres of 0x0F000000. 
Note that kernel may allocate memory for other processes in that range and that is the primary reason to write a kernel module which would request_mem_region so no other processes would overlap with the memory range.
Besides reserving memory ranges the kernel module provides a sanitized way of accessing the hardware from userspace applications via /dev/blah block devices.
```

I would like to add that if you want to make a [continuous transfers](https://github.com/DamienFruleux/ZCU111-Doc/blob/main/1-AXI_DMA.md#use-a-fifo), in order to get rid of the [DMA limit of 67 108 863 bytes](https://github.com/DamienFruleux/ZCU111-Doc/blob/main/1-AXI_DMA.md#maximum-size-of-a-transfert), you must repeat the 4 steps in a loop.

## Use PLRAM

With the right memory addresses, It works perfectly and without any problems because it is not used by the Linux Kernel.

## Use PSRAM

### 1- Use "LAURI" example

With the right memory addresses, It works but you have to be careful, because this memory space is not reserved and can be used by the kernel ! This can be interesting to do some tests, but should not be used in a project!

### 2- Use kernel module

As Lauri explained earlier, you have to be careful with the use of memory. In fact, with this example, we use a memory area without in a "dirty" way. In fact, it would be necessary to develop a kernel module that would reserve this memory area, in order to be sure that the kernel cannot use it for other processes. 

I resume the explanations of [Thomas Petazzoni](https://stackoverflow.com/a/7738534), which are in my opinion perfect: 

```
request_mem_region tells the kernel that your driver is going to use this range of I/O addresses, which will prevent other drivers to make any overlapping call to the same region through request_mem_region.
This mechanism does not do any kind of mapping, it's a pure reservation mechanism, which relies on the fact that all kernel device drivers must be nice, and they must call request_mem_region, check the return value, and behave properly in case of error.
```

I haven't had time to make a kernel module for this yet. But I might do it one day.

### 3- Use kernel module to allocate memory

first **reserv memory** and **allocate memory** next

### 4- Use Dynamic memory allocation

I would like to point out that it is surely possible to do a dynamic allocation (*malloc* function) and to use the address of the allocated memory space.

However, although this avoids programming in the kernel, one cannot be sure that the memory is continuously allocated, which could be problematic as we have already explained

I haven't tested this theory yet, maybe I'll do it sometime. 

### 5- Use Dynamic memory allocation in a kernel module

It is also possible to use dynamic memory allocation in the kernel, using the *kmalloc* function. This solves the problems related to memory allocation, in addition to allowing **continuous** memory allocation!

Also, I haven't investigated this possibility yet

### Use more than 4 GB SODIMM RAM

#### Linux always "see" 4GB of RAM

Indeed, by default the ZCU111 has a 4GB SODIMM. If we use the method described above, we run the risk of using memory addresses potentially used by the Linux Kernel. As I needed a lot of memory and not to use dynamic allocation (data continuity), I upgraded the SODIMM up to the maximum possible (32GB) and I made sure that it is well mapped in /dev/memory, but without the Linux kernel being able to exploit it like classic RAM. More simply, linux still sees 4GB of RAM, but can access the other 28GB of the PS in the same way as the 4GB of the PL. 

You can find more information on this subject [here] (in progress)

#### Linux see all 32GB of RAM

Note however that it is possible to use the 32GB of RAM with the Linux Kernel and to allocate the necessary amount of memory dynamically. I cannot however certify the continuity of the data in memory. (Note that with a kernel modulator and the kmalloc function this seems possible). 

Finally, this "method" allows me to use a single code to use the PS and the PL, since only the memory address space differs. 

# Use C program instead of python program

Since C is a compiled language and Python is an interpreted language, it is sometimes necessary to use the first one. In particular if we want to be sure that the different instructions (for the control of the DMA) located in the loop (which allows to make a continuous transfer) are executed quickly enough, it is perhaps necessary to pass by a language closer to the machine.

In particular it is necessary to make sure that the data is sent fast enough to the DMA, in order to be sure to have a continuous signal to send to the DAC.