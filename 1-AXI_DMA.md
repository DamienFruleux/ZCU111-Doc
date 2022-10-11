# Definition of DMA

Literally, a [Direct Memory Access](https://en.wikipedia.org/wiki/Direct_memory_access) (DMA) is a device that allows direct access between the RAM and a device (e.g. HDD, SSD), without the intervention of the processor, except to start and stop the transfer. 

At Xilinx, we find DMAs with the same definition, but which allow more precisely to transfer data between an AXI4 (Memory Map) interface (which is therefore connected to the SODIMM RAM, or **PS RAM**) and an AXI4-Stream interface (which is therefore connected to an **external device**, such as a DAC or ADC).

Note that it is also possible to use a external (to the processor) RAM, in particular the 4 GB of DDR4 SDRAM, or **PL RAM**. It also uses an AXI4 (Memory Map) interface.

In particular, the DMA has 2 separate channels: 

- Read Channel : **read data from RAM** and write it directly to the device.

- Write Channel : read data from the device and **write it directly to the RAM**.

Finally, the [AXI DMA Controller](https://www.xilinx.com/products/intellectual-property/axi_dma.html) Xilinx IP allows you to transfer data between the Processing System (ARM processors) and the Programable Logical (FPGA). More explanations on the use with the PS and details on the use of DAC/ADC will be given next.

# AXI4 Protocol

The [Advanced eXtensible Interface 4](https://en.wikipedia.org/wiki/Advanced_eXtensible_Interface) (AXI4) is a parallel communication interface, mainly designed for on-chip communication.

There are three types of AXI4 interfaces [UG761, page 4](https://docs.xilinx.com/v/u/en-US/ug761_axi_reference_guide) :

- **AXI4** : for Memory Mapped (MM) interfaces. **It allows burst of up to 256 data transfer cycles with just a single address phase**. Useful for high-performance requirements. 

- **AXI4-Lite** : light-weight, **single transaction memory mapped interface**. It has a small logic footprint and is a simple interface to work with both in design and usage. Useful for simple, low-throughput memory-mapped communication (for example, to and from control and status registers).

- **AXI4-Stream** : **removes the requirement for an address phase altogether and allows unlimited data burst size**. AXI4-Stream interfaces and transfers do not have address phases and are therefore not considered to be memory-mapped. Useful for high-speed streaming data (**I like to talk about a "real time transfer" to illustrate the difference with AXI4 (MM)**).

In practice, for the AXI4 (MM) and AXI4-Lite interfaces, it is necessary to read or write in the registers provided at right addresses. For the AXI4-Stream interface, it is necessary to use a DMA.

# Technical Characteristics of DMA

![DMA](./images/DMA.png?raw=true "AXI DMA Controller Xilinx IP")

## Maximum size of a transfert

The maximum width of buffer length register is **2<sup>26</sup>-1 bytes**. This means that a DMA transfert cannot contain more than **64 MB - 1 (or 64 Mo - 1)** of data (this is a hardware limit of the Xilinx IP).

> More precisely : 2<sup>26</sup> bits = 67 108 864 b = 64 Mb = 8 MB (or 8 Mo)

It is important to know this because basically, we cannot make a DMA transfer of more than 64 MB-1, in particular if we use the Pynq library. A method to bypass this limit is proposed in the following.

Be careful, vivado is not very clear in the configuration, because it requires a number of bits  :  We can read in [PG021](https://docs.xilinx.com/r/en-US/pg021_axi_dma/Width-of-Buffer-Length-Register) : 
*The number of bytes is equal to 2 Length Width . So a Length Width of 26 gives a byte count of 67,108,863 bytes.* 

## Maximum speed of a transfert

### AXI4 (MM) side, usage of the PS RAM

On the AXI4 (MM) side, the speed is limited by the PS : the AXI4 HP0/1/2/3 data width is limited to **128 bits**.
Considering that maximum clock frequency of the PS is **333 MHz**, the maximum transfert rate on the AXI4 (MM) side is **41.625 Gbps = 5.20 GB/s (or 5.20 Go/s)**.

> More precisely : 128 bits * 333 MHz = 42 624 x 10<sup>6</sup> bps = 41.625 Gbps = 5.20 GB/s (or 5.20 Go/s)

> With a more regular 300 MHz clock :  128 bits * 300 MHz = 38 400 x 10<sup>6</sup> bps = 37.5 Gbps = 4.6875 GB/s (or 4.6875 Go/s)

Moreover, HP1 and HP2 seem to share the same 128 bits AXI MM interface  [UG1085, figure 1-1, page 29](https://www.xilinx.com/support/documentation/user_guides/ug1085-zynq-ultrascale-trm.pdf), so the speed will not be maximal if they are both used at the same time.

#### Using Samples per seconds

Later, we will use a DMA to send or receive data with the DACs and ADCs. In general, it is more convenient to speak in GSa/s (Samples per seconds, or GSPS in Xilinx nomenclature) instead of GB/S (or Go/s). 

As we will see, DACs and ADCs use data coded on 16 bits, so **1 sample = 16 bits = 2 Bytes (or 2 octets)**.

The maximum transfer rate on this side is also **2.6 GSa/s**, or **2.34375 GSa/s** with a more regular 300 MHz clock.

### AXI4 (MM) side, usage of the PL RAM

As we have seen above, it is also possible to use the 4 GB of DDR4 SDRAM. In this case, the data width is limited to **512 bits**. 
Moreover, the DDR4 SDRAM memory need a clock frequency of **300 MHz**. In this case, the maximum transfert rate on the AXI4 (MM) side is **150 Gbps = 18.75 GB/s (or 18.75 Go/s)**. 

> More precisely : 512 bits * 300 MHz = 153 600 x 10<sup>6</sup> bps = 150 Gbps = 18.75 GB/s (or 18.75 Go/s)

#### Using Samples per seconds

The maximum transfer rate on the AXI4 (MM) side is also **9.375 GSa/s**.

### AXI4-Stream side

According to [PG021](https://docs.xilinx.com/r/en-US/pg021_axi_dma), AXI4-Stream data width support of 8, 16, 32, 64, 128, 256, 512 and 1024 bits. Tt is the number of bits that will be sent or received by the DMA in 1 clock stroke, but it does not correspond to the total amount of data transferred by the DMA, which is 64 MB - 1 ( or 64 Mo - 1), as seen before.

On the AXI4-Stream side, the speed is limited by the Intellectual Property (IP) that you use : generally, it is not possible to choose the parameters, except if you develop your own IP. This is what we will do next. In our example, we will use data width of 128 or 256 bits depending on the case (ADCs or DACs)

## Maximum amount of RAM

According to [DS889, page 12](https://docs.xilinx.com/v/u/en-US/ds889-zynq-usp-rfsoc-overview) it is possible to upgrade the ZCU111 SODIMM RAM, in order to have up to **32 GB (or 32 Go) of memory in the PS, of which 28GB is actually usable**, which corresponds to **14 GSa in the PS**. We will explain next how to process to RAM upgrade, and why we only have 28GB usable. 

But unfortunately, it is not possible to increase the DDR4 SDRAM, because this memory is soldered, there is only **4 GB (or 4 Go) of memory in the PL**, which corresponds to **2 GSa of samples in the PL**. 

It is necessary to choose between speed and amount of memory : 

*28 GB @ 5.20 GB/s vs 4 GB @ 18.75 GB/s* !

## Conclusion

As we have just seen, the upstream part of the DMA works with the AXI4-Memory Map protocol, which is a protocol that works by packets, while the downstream part works with the AXI4-Stream protocol, which is a protocol that works by a data flow. The 2 protocols are different and it is thus the DMA which makes the connection. 

# Use a FIFO

## Utility

In order to adapt the frequency, we will use a FIFO with 2 independent clocks which will allow us to play the role of buffer. The FIFO is filled at a frequency, and emptied at an other frequency. The FIFO makes the link between the 2 frequencies by temporarily storing the data : it allow to "synchronize" the data between 2 interfaces having different clocks.

## Definition of a FIFO

A [FIFO](https://en.wikipedia.org/wiki/FIFO_(computing_and_electronics)) (first in, first out : the first in is the first out) is a method for organising the manipulation of a data buffer, where the oldest (first) entry, is processed first. 

The [AXI Streaming FIFO](https://www.xilinx.com/products/intellectual-property/axi_fifo.html) Xilinx IP which will be use.

## Technical Characteristics of FIFO

![FIFO](./images/FIFO.png?raw=true "AXI Streaming FIFO Xilinx IP")

### FIFO Size

According to [PG085](https://www.xilinx.com/support/documents/ip_documentation/axis_infrastructure_ip_suite/v1_0/pg085-axi4stream-infrastructure.pdf) FIFO has a maximum memory depth of 32 768 (this is a hardware limit of the Xilinx IP). 

- For an 128 bits AXI4-Stream data width, this corresponds to **512 KB (or 512 Ko)**.

> More precisely : 32 768 * 128 bits = 4 194 304 bits = 4 Mb = **512 KB (or 512 Ko)**

- For an 256 bits AXI4-Stream data width, this corresponds to **1 MB (or 1 Mo)**.

> More precisely : 32 768 * 256 bits = 8 388 608 bits = 8 Mb = **1 MB (or 1 Mo)**

### Independent Clock

Be sure to choose this option to be able to read and write in the FIFO with 2 different frequencies
