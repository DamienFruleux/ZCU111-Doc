# 1- AXI4 Protocol

The AXI4 ([Advanced eXtensible Interface 4](https://en.wikipedia.org/wiki/Advanced_eXtensible_Interface)) is a parallel communication interface, mainly designed for on-chip communication.

There are three types of AXI4 interfaces ([UG761](https://docs.xilinx.com/v/u/en-US/ug761_axi_reference_guide)) :

- **AXI4 Memory Map** : for Memory Mapped Memory Map interfaces.
**It allows burst of up to 256 data transfer cycles with just a single address phase**.
Useful for high-performance requirements. 

- **AXI4-Lite** : light-weight, **single transaction memory mapped interface**.
It has a small logic footprint and is a simple interface to work with both in design and usage.
Useful for simple, low-throughput memory-mapped communication (for example, to and from control and status registers).

- **AXI4-Stream** : **removes the requirement for an address phase altogether and allows unlimited data burst size**.
AXI4-Stream interfaces and transfers do not have address phases and are therefore not considered to be memory-mapped.
Useful for high-speed streaming data (I like to talk about "real time transfer" to illustrate the difference with packet transfers like AXI4 Memory Map).

In practice, for the AXI4 Memory Map and AXI4-Lite interfaces, it is necessary to read or write to the configuration registers at the correct addresses.
To operate the AXI4-Stream interface, a DMA must be used.

# 2- Definition of DMA

Literally, a DMA ([Direct Memory Access](https://en.wikipedia.org/wiki/Direct_memory_access)) is a device that allows direct access between the RAM and a device (e.g. HDD, SSD), without the intervention of the processor, except to start and stop the transfer. 
This speeds up data transfer considerably.

At Xilinx, we find DMAs with the same definition, but which allow more precisely to transfer data between an AXI4 Memory Map interface (which is therefore connected to the SODIMM RAM, or **PS RAM**) and an AXI4-Stream interface (which is therefore connected to an **external device**, such as a DAC or ADC).

Note that it is also possible to use a external RAM, in particular the 4 GB of DDR4 SDRAM, or **PL RAM**.
It also uses an AXI4 Memory Map interface.

For the rest of these explanations, we'll consider only the Xilinx DMA.

In particular, the DMA has 2 separate channels: 

- Read Channel : **read data from RAM** and write it to the device.

- Write Channel : read data from the device and **write it to the RAM**.

The IP (Intellectual Property) [AXI DMA Controller](https://www.xilinx.com/products/intellectual-property/axi_dma.html) supplied by Xilinx is used to transfer data between the PS (Processing System: ARM CPU) and the PL (Programable Logical: FPGA).

# 3- Technical Characteristics of DMA

![DMA](./../images/DMA.png?raw=true "AXI DMA Controller Xilinx IP")

## A- Maximum size of a transfert

The maximum width of buffer length register is **2<sup>26</sup>-1 bytes**.
This means that a DMA transfert cannot contain more than **64 MB - 1** of data (this is a hardware limit of the Xilinx IP).

> More precisely : 2<sup>26</sup> bytes = 67 108 864 B = 64 MB

It is important to know this because basically, we cannot make a DMA transfer of more than **64 MB - 1**, in particular if we use the Pynq library.
A method to bypass this limit is proposed in the following.

Be careful, vivado is not very clear in the configuration, because it requires a number of bits ([PG021](https://docs.xilinx.com/r/en-US/pg021_axi_dma)) : "*The number of bytes is equal to **2<sup>Length Width</sup>** . So a Length Width of 26 gives a byte count of **67,108,863 bytes**.* "

## B- Maximum speed of a transfert

### 1-AXI4 Memory Map side, usage of the PS RAM

On the AXI4 Memory Map side, the speed is limited by the PS : the AXI4 HP0/1/2/3 data width is limited to **128 bits**.
Considering that maximum clock frequency of the PS is **333 MHz**, the maximum transfert rate on the AXI4 Memory Map side is **5.20 GB/s**.

> More precisely : 128 bits * 333 MHz = 42 624 * 10<sup>6</sup> bps = 41.625 Gbps = 5.20 GB/s

With a more conventional **300 MHz** clock, a transfer rate of **4.6875 GB/s** is achieved.

> More precisely : 128 bits * 300 MHz = 38 400 * 10<sup>6</sup> bps = 37.5 Gbps = 4.6875 GB/s

Moreover, HP1 and HP2 share the same 128 bits AXI4 Memory Map interface ([UG1085](https://www.xilinx.com/support/documentation/user_guides/ug1085-zynq-ultrascale-trm.pdf)), so the speed will not be maximal if they are both used at the same time.

#### Using Samples per seconds

Later, we will use a DMA to send or receive data with the DACs and ADCs.
In general, it is more convenient to speak in GSa/s (Samples per seconds, or GSPS in Xilinx nomenclature) instead of GB/S. 

As we will see, DACs and ADCs use data coded on 16 bits, so **1 sample = 16 bits = 2 Bytes**.

The maximum transfer rate on this side is also **2.6 GSa/s** with a 333 MHz clock, or **2.34375 GSa/s** with a more regular 300 MHz clock.

It's preferable to use a 300 MHz clock, because as well as being more conventional, this is the frequency at which the PL RAM operates: this makes it easy to reuse the same design between the PS RAM and the PL RAM, thus increasing compatibility.
If maximum performance is required, there's still a 33 MHz margin.

### 2-AXI4 Memory Map side, usage of the PL RAM

As we have seen above, it is also possible to use the 4 GB of DDR4 SDRAM (PL RAM).
In this case, the data width is **512 bits**. 
Moreover, the DDR4 SDRAM memory need a clock frequency of **300 MHz**.
In this case, the maximum transfert rate on the AXI4 Memory Map side is **18.75 GB/s**. 

> More precisely : 512 bits * 300 MHz = 153 600 * 10<sup>6</sup> bps = 150 Gbps = 18.75 GB/s

#### Using Samples per seconds

The maximum transfer rate on the AXI4 Memory Map side is also **9.375 GSa/s**.

### 3-AXI4-Stream side

The data width of the AXI4-Stream DMA interface can be 8, 16, 32, 64, 128, 256, 512 and 1024 bits ([PG021](https://docs.xilinx.com/r/en-US/pg021_axi_dma)).
It is the number of bits that will be sent or received by the DMA in 1 clock stroke, but it does not correspond to the total amount of data transferred by the DMA, which is 64 MB - 1 ( or 64 Mo - 1), as seen before.

On the other side of the interface, speed is defined by the IP used: generally, it is not possible to choose this parameter directly, as it depends on the operation of this IP.
In our case, we'll be using data widths of 128, 256, 512 or 1024 bits, depending on the case (ADCs or DACs) and the number of channels (1 or 2).

## C- Maximum amount of memory

The ZCU111 has 2 different RAMs: PS RAM and PL RAM.

It is possible to change the SODIMM RAM strip (([DS889](https://docs.xilinx.com/v/u/en-US/ds889-zynq-usp-rfsoc-overview))), in order to upgrade from 4 GB (default) to **32 GB of memory in the PS**.
In theory, this corresponds to 2 GSa and 16 GSa respectively, but bear in mind that the GNU/Linux kernel also uses this memory!
The amount of memory that can actually be used is therefore not perfectly defined.
Finally, you can't manage this memory as you wish: you have to allocate memory.

Unfortunately, it's not possible to increase DDR4 SDRAM, as these memory chips are soldered.
The PL has **4 GB of memory**.
What's more, this memory is not used at all by the kernel, so we have 2 GSa available that we can use and manage as we see fit.

So there's a trade-off between speed and memory: **~ 32 GB @ 5.20 GB/s or 4 GB @ 18.75 GB/s !**

## D- Conclusion

As we have just seen, the upstream part of the DMA uses the AXI4 Memory Map protocol, which is a protocol that works by packets, while the downstream part uses the AXI4-Stream protocol, which is a protocol that works by a data flow.
The 2 protocols are different and it is thus the DMA which makes the connection. 

# 4- Use a FIFO

## A- Utility

In order to adapt the frequency, we will use a FIFO with 2 independent clocks which will allow us to play the role of buffer.
The FIFO is filled at a frequency, and emptied at an other frequency.
The FIFO makes the link between the 2 frequencies by temporarily storing the data : it allow to "synchronize" the data between 2 interfaces having different clocks.

## B- Definition of a FIFO

A [FIFO](https://en.wikipedia.org/wiki/FIFO_(computing_and_electronics)) (first in, first out : the first in is the first out) is a method for organising the manipulation of a data buffer, where the oldest (first) entry, is processed first. 

The [AXI Streaming FIFO](https://www.xilinx.com/products/intellectual-property/axi_fifo.html) Xilinx IP which will be use.

## C- Technical Characteristics of FIFO

![FIFO](./images/FIFO.png?raw=true "AXI Streaming FIFO Xilinx IP")

### 1-FIFO Size

According to [PG085](https://docs.xilinx.com/r/en-US/pg085-axi4stream-infrastructure) FIFO has a maximum memory depth of 32 768 (this is a hardware limit of the Xilinx IP). 

- For an 128 bits AXI4-Stream data width, this corresponds to **512 KB**.

> More precisely : 32 768 * 128 bits = 4 194 304 bits = 4 Mb = **512 KB**

- For an 256 bits AXI4-Stream data width, this corresponds to **1 MB**.

> More precisely : 32 768 * 256 bits = 8 388 608 bits = 8 Mb = **1 MB**

- For an 512 bits AXI4-Stream data width, this corresponds to **2 MB**.

> More precisely : 32 768 * 512 bits = 16 777 216 bits = 16 Mb = **2 MB**.

- For an 256 bits AXI4-Stream data width, this corresponds to **4 MB**.

> More precisely : 32 768 * 1024 bits = 33 554 432 bits = 32 Mb = **4 MB**

### 2-Independent Clock

Be sure to choose this option to be able to read and write in the FIFO with 2 different frequencies.


### 3-Continuous transfert

Thanks to this FIFO, which acts as a buffer, we also have the possibility of making continuous transfers. 

In fact, if the memory depth is sufficient, it is possible to launch a DMA transfer and before all the data is transferred to the AXI4-Stream side, it is possible to restart a DMA transfer and so on. 

In this way, the data is transferred continuously over time, without any "gaps" in the signal. 

It is therefore possible to artificially increase the size of a DMA transfer to exceed the limit of 67 108 863 bytes (or 64MB - 1).