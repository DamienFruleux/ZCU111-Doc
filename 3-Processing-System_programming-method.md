# 1- Introduction

There are different ways to proceed for the "data management". 

The easiest way is certainly to use a pynq notebook and the [dma class of the Pynq library](https://pynq.readthedocs.io/en/latest/pynq_libraries/dma.html), but this has some limitations.

That's why I developed a C code that does almost the same thing, but in a lower level, using the standard POSIX libraries ([example 1](https://lauri.xn--vsandi-pxa.com/hdl/zynq/xilinx-dma.html) and [example 2](https://www.hackster.io/whitney-knitter/introduction-to-using-axi-dma-in-embedded-linux-5264ec)). This allows us to have more flexibility on the various actions to be carried out. Of course, it is possible to do the same thing in another programming language, like Python.

If necessary, it is possible to configure and obtain a certain number of parameters in the different IPs, for example the frequency of the PLLs for the DACs, the state of the DMAs... This configuration can be done easily with the Pynq libraries because most of the functions are implemented, which is not necessarily the case in the proposed C program, which only contains the elements necessary for proper operation. It is not impossible that I implement all this later. 

For reasons of simplicity, we preferred to continue to load the overlay using the "Overlay" function of the Pynq library, even with the C language codes. We are currently working on a better understanding of these principles in order to do without the Pynq library in the future. 

As there are 2 types of memories (PS and PL) that can be used in each case, I will detail the different possibilities. Be careful though, not all possibilities are possible, especially with Pynq.

# 2- General operation of the Vivado design

The operating principle is relatively simple and remains the same in the different cases.

For the DACs, the steps are as follows: 
- 1: we load the bitstream (the desing of the circuit)
- 2: we place in memory the samples we want to send
- 3: we launch a DMA transfer to send the data in memory to the DACs

For the ADCs, the steps are as follows: 
- 1: we load the bitstream (the desing of the circuit)
- 2: we launch a DMA transfer in order to receive the data in memory from the ADCs
- 3: we recover the samples received from the memory


# 3- Use Pynq libraries

As I explained, the pynq_notebook uses the Pynq libraries. Although it works perfectly, there are some limitations that can become annoying in some cases. 

## A- Program Operation

The explanations of [dma class of the Pynq library](https://pynq.readthedocs.io/en/latest/pynq_libraries/dma.html) are very well explained and illustrate very well the different steps required.

We use the functions available in the Pynq libraries to carry out the transfer. The code can be summarized in the following steps: 

- 1: load the bitstream in the memory (*Overlay* function)
- 2: reserve the necessary amount of memory (*allocate* function, itself based on the **cma library which allows a dynamic allocation**)
- 3: fill the reserved memory with the samples to be converted (for example: 1 sine and 1 cosine)
- 4: launch the DMA transfer (*sendchannel.transfer* and *sendchannel.wait* functions)

We can present the associated piece of code:

```
import numpy as np
from pynq import allocate
from pynq import Overlay

ol = Overlay('example.bit')
dma = ol.axi_dma

input_buffer = allocate(shape=(5,), dtype=np.uint16)
output_buffer = allocate(shape=(5,), dtype=np.uint16)

for i in range(5):
   input_buffer[i] = i

dma.sendchannel.transfer(input_buffer)
dma.sendchannel.wait()

```

It works perfectly. However, there are a number of limitations.

## B- Limits with PSRAM

### 1-Continuous transfert

Note that if it is possible to make continuous transfers in C (explained below), Pynq does not yet offer this method directly (to my knowledge).

I have previously how to perform [continuous transfers](https://github.com/DamienFruleux/ZCU111-Doc/blob/main/1-AXI_DMA.md#continuous-transfert), in order to get rid of the [DMA limit of 67 108 863 bytes (or 64MB - 1)](https://github.com/DamienFruleux/ZCU111-Doc/blob/main/1-AXI_DMA.md#maximum-size-of-a-transfert). However, due to the very high level operation of the pynq functions, it is almost certain that the time between the end of the execution of the 1st function *dma.sendchannel.transfer(input_buffer_a)* and the time between the beginning of the 2nd function *dma.sendchannel.transfer(input_buffer_b)* will be probably too long to allow a continuous transfer. I haven't tested this theory yet, maybe I'll do it sometime. In this case, we would need a FIFO with a memory depth large enough to act as a buffer long enough.

While it may work, it doesn't allow us to manage our transfers accurately enough. 

### 2-Buffer size

By default, it is not possible to allocate more than 128MB of data using the *allocate* function provided by pynq : [This buffer is allocated inside the kernel space using xlnk driver. The maximum allocatable memory is defined at kernel build time using the CMA memory parameters. For Pynq-Z1 kernel, it is specified as 128MB.](https://pynq.readthedocs.io/en/v2.0/_modules/pynq/xlnk.html)

This documentation is from version 2.0 of pynq, it is possible that the size has increased, especially for the ZCU111, but in any case, the principle remains the same: there is a significant size limit.

In order to extend this limitation, it is necessary to compile the Kernel by modifying some options, in particular : 

- Device Drivers->Generic Driver Options->Size in Mega Bytes(1024)

- Device Drivers->Staging drivers (ON)->Xilinx APF Accelerator driver (ON)->Xilinx APF DMA engines support (ON) 

You will find more information [here](https://discuss.pynq.io/t/pynq-maximum-allocatable-memory-cma/1593), but I intend to add some documentation on this task later.

Although relatively simple, it requires additional work and does not necessarily correspond to our specifications.

### 3-Dynamic memory allocation

Pynq uses dynamic memory allocation (malloc function in C), which means that it asks the kernel to allocate a certain amount of memory. So there are limits on the amount of memory allocated, since the OS must keep a certain amount of memory: it is not possible to allocate all 4GB of RAM to store data ! Even though the amount of memory that can be used is considerably increased with the cma library.

### 4-Data arrangement in memory

Finally, even if it is possible to allocate the desired amount of memory after some manipulations, it is not certain that the data are arranged continuously in memory, especially if there are large amounts of data, which could potentially cause some problems since **the analog signal is continuous over time** : we could then see artifacts appear in the analog signal, like holes.
The DMA can use a mechanism called SG for this. It will be necessary to make sure that the FIFO memory depth is large enough to allow time for the DMA to "change memory area". However, this point is more of a question than a statement. 

## C- Limits with PLRAM

With Pynq, to use the PLRAM, the [mmio](https://pynq.readthedocs.io/en/v2.6.1/pynq_libraries/mmio.html) class must be used. In particular, to write data, you must use the function **mmio.write(address_offset, data)**.

The operating principle of the [dma](https://pynq.readthedocs.io/en/v2.6.1/pynq_libraries/dma.html) class requires that the **dma.sendchannel.transfer()** function takes an input_buffer as argument. 

These particularities have not yet allowed me to use PLRAM as memory to store data before sending them to the DACs (or after receiving them from the ADCs), with Pynq.

Finally, although it seems possible, the documentation does not seem to consider this possibility as interesting : [MMIO provides a simple but powerful way to access and control peripherals. For simple peripherals with a small number of memory accesses, or where performance is not critical, MMIO is usually sufficient for most developers. If performance is critical, or large amounts of data need to be transferred between PS and PL, using the Zynq HP interfaces with DMA IP and the PYNQ DMA class may be more appropriate](https://pynq.readthedocs.io/en/v2.6.1/pynq_libraries/mmio.html).

If you have any information, don't hesitate to share it with me.

# 4- Use C standard libraries

In order to overcome these limitations, we propose not to use the Pynq library anymore, but to use lower level functions. In particular, I have chosen to use the C language to initiate DMA transfers by directly using the control registers available in [PG021](https://docs.xilinx.com/r/en-US/pg021_axi_dma): we can perform continuous transfers of all available memory, and not be limited to a single DMA transfer.

## A- Program Operation

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

We use the functions available in the standard POSIX libraries to carry out the transfer. The code can be summarized in the following steps: 

- 1: load the bitstream in the memory (as explained at the beginning, we continue to do this step with Pynq tools for the moment)
- 2: fill the memory range with the samples to be converted (for example: 1 sine and 1 cosine)
- 3: launch the DMA transfer (explained in the quote above)

We can present the associated piece of code:

```
// File Descriptor : open /dev/mem which represents the whole physical memory
int fd = open("/dev/mem", O_RDWR | O_SYNC);

// Memory Map Source Address
int16_t* mem = mmap(NULL, number_octet, PROT_READ | PROT_WRITE, MAP_SHARED, fd, MEMORY_ADDR);

// Memory Map DMA (AXI Lite register block)
int32_t* dma = mmap(NULL, REGISTER_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, DMA_ADDR);

// Calculate & saving data in memory
for (uint64_t i=0; i<number_samples;i+=2){
	mem[i] = (int16_t)(pow(2,14)*sin(i/2*3.14/180));
	mem[i+1] = (int16_t)(pow(2,14)*cos(i/2*3.14/180));
}

// Send data from memory to DMA

// Reset DMA
dma[MM2S_CONTROL_REGISTER] = 0x4;

for(unsigned int n=0; n<number_packet; n++){
	// Halt DMA
	dma[MM2S_CONTROL_REGISTER] = 0x0;

	// Write source addresses
	dma[MM2S_START_ADDRESS_LSB] = MEMORY_ADDR_LSB+n*DMA_SIZE;
	dma[MM2S_START_ADDRESS_MSB] = MEMORY_ADDR_MSB;

	// Start DMA
	dma[MM2S_CONTROL_REGISTER] = 0x1;

	// Begin tranfer
	dma[MM2S_LENGTH] = DMA_SIZE;

	// Wait until done
	while (((dma[MM2S_STATUS_REGISTER] >> 1) & 0x00000001)!= 1) {}
}

// Munmap & Close
munmap((void *) dma, REGISTER_SIZE);
munmap((void *) mem, number_octet);
close(fd);

```
We notice that we do not use dynamic memory allocation in this part, but we read and write directly to the memory areas : it should be used with knowledge (see below for more information about this).
However, note that kernel may allocate memory for other processes in this area because this memory space is not reserved. Also, it is possible to delete important information saved by the kernel to these memory addresses. 

It works perfectly. This time without limitations if you use the right memory addresses, but with more knowledge needed!

## B- Use PLRAM

With the right memory addresses, It works perfectly and without any problems because it is not used by the Linux Kernel. Please refer to [UG1085](https://docs.xilinx.com/r/en-US/ug1085-zynq-ultrascale-trm) for more information about memory address spaces. 

## C- Use PSRAM

### 1-Use "LAURI" example

As already explained, it works but you have to be careful, because this memory space is not reserved and can be used by the kernel ! This can be interesting to do some tests, but should not be used in a project! There are different ways to use the memory more appropriately.

### 2-Use kernel module to request memory region

As Lauri explained earlier, you have to be careful with the use of memory. In fact, with this example, we use a memory area without in a "dirty" way. In fact, it would be necessary to develop a kernel module that would reserve this memory area, in order to be sure that the kernel cannot use it for other processes. 

I resume the explanations of [Thomas Petazzoni](https://stackoverflow.com/a/7738534), which are in my opinion perfect: 

```
request_mem_region tells the kernel that your driver is going to use this range of I/O addresses, which will prevent other drivers to make any overlapping call to the same region through request_mem_region.
This mechanism does not do any kind of mapping, it's a pure reservation mechanism, which relies on the fact that all kernel device drivers must be nice, and they must call request_mem_region, check the return value, and behave properly in case of error.
```

I haven't had time to make a kernel module for this yet. But I might do it one day.

### 3-Use kernel module to allocate memory

First **reserv memory** and **allocate memory** next

TODO : need to delete ?

### 4-Use Dynamic memory allocation

I would like to point out that it is surely possible to do a dynamic allocation (*malloc* function) and to use the address of the allocated memory space.

However, although this avoids programming in the kernel, one cannot be sure that the memory is continuously allocated, which could be problematic as we have already explained

I haven't tested this theory yet, maybe I'll do it sometime. 

### 5-Use Dynamic memory allocation in a kernel module

It is also possible to use dynamic memory allocation in the kernel, using the *kmalloc* function. This solves the problems related to memory allocation, in addition to allowing **continuous** memory allocation!

Also, I haven't investigated this possibility yet

## D- Use more than 4 GB SODIMM RAM

We are limited to 4GB of RAM, without counting the memory needed to run the linux kernel and the operating system. We would like to use as many samples as possible.
As explained above, it is possible to change the SODIMM ZCU111 RAM strip, in order to have 32 GB (or 32 GB) of memory in the PS.

### 1-Linux see all 32GB of RAM

Note, however, that it is possible to use the 32 GB of RAM with the Linux kernel and dynamically allocate the amount of memory needed.

This is the traditional use of RAM, so it is much more flexible because the kernel chooses and uses the 32 GB of RAM as it wishes. However, it is necessary to manage to allocate the memory as explained in the previous points. 

Moreover, I cannot certify the continuity of the data in memory. (Note that with a kernel module and the kmalloc function this seems possible). 

### 2-Linux always "see" 4GB of RAM

Indeed, by default the ZCU111 has a 4GB SODIMM. If we use the method described above, we run the risk of using memory addresses potentially used by the Linux Kernel. As I needed a lot of memory and not to use dynamic allocation (data continuity and complexity), I upgraded the SODIMM up to the maximum possible (32GB) and I made sure that it is well mapped in /dev/memory, but without the Linux kernel being able to exploit it like classic RAM. More simply, linux still sees 4GB of RAM, but can access the other 28GB of the PS in the same way as the 4GB of the PL. As the samples are coded on 16 bits (=2 bytes or 2 octets), it is possible to store up to 14 GSa in this unallocated memory for the Kernel.  

You can find more information on this subject later (in progress). 

Finally, this "method" allows me to use a single code to use the PS and the PL, since only the memory address space differs. 

# 5- Use C program instead of python program

Since C is a compiled language and Python is an interpreted language, it is sometimes necessary to use the first one. In particular if we want to be sure that the different instructions (for the control of the DMA) located in the loop (which allows to make a continuous transfer) are executed quickly enough, it is perhaps necessary to pass by a language closer to the machine.

In particular it is necessary to make sure that the data is sent fast enough to the DMA, in order to be sure to have a continuous signal to send to the DAC.
