# Definition

Literally, a [Direct Memory Access](https://en.wikipedia.org/wiki/Direct_memory_access) (DMA) is a device that allows direct access between the RAM and a device (e.g. HDD, SSD), without the intervention of the processor, except to start and stop the transfer. 

At Xilinx, we find DMAs with the same definition, but which allow more precisely to transfer data between an AXI Memory Map interface (which is therefore connected to the SODIMM RAM, or **PS RAM**) and an AXI Stream interface (which is therefore connected to an **external device**, such as a DAC or ADC).

Note that it is also possible to use a external (to the processor) RAM, in particular the 4 GB of DDR4 SDRAM, or **PL RAM**. It also uses an AXI Memory Map interface.


In particular, the DMA has 2 separate channels: 

- Read Channel : **read data from RAM** and write it directly to the device.

- Write Channel : read data from the device and **write it directly to the RAM**.

# AXI4 Protocol

The [Advanced eXtensible Interface](https://en.wikipedia.org/wiki/Advanced_eXtensible_Interface) (AXI) is a parallel communication interface, mainly designed for on-chip communication.

There are three types of AXI4 interfaces [UG761, page 4](https://www.xilinx.com/support/documentation/ip_documentation/axi_ref_guide/latest/ug761_axi_reference_guide.pdf) :

- **AXI4** : for Memory Mapped (MM) interfaces. **It allows burst of up to 256 data transfer cycles with just a single address phase**. Useful for high-performance requirements. 

- **AXI4-Lite** : light-weight, **single transaction memory mapped interface**. It has a small logic footprint and is a simple interface to work with both in design and usage. Useful for simple, low-throughput memory-mapped communication (for example, to and from control and status registers).

- **AXI4-Stream** : **removes the requirement for an address phase altogether and allows unlimited data burst size**. AXI4-Stream interfaces and transfers do not have address phases and are therefore not considered to be memory-mapped. Useful for high-speed streaming data (**I like to talk about a "real time transfer" to illustrate the difference with AXI4-MM**).

In practice, for the AXI4-MM and AXI4-Lite interfaces, it is necessary to read or write in the registers provided at right addresses. For the AXI4-S interface, it is necessary to use a DMA.

# Technical Characteristics of DMA

## Maximum size of a transfert

The maximum width of buffer length register is **2<sup>26</sup>-1 bytes**. This means that a DMA transfert cannot contain more than **64 MB - 1 (or 64 Mo - 1)** of data (this is a hardware limit of the Xilinx IP).

> More precisely : 2<sup>26</sup> bits = 67 108 864 b = 64 Mb = 8 MB (or 8 Mo)

Nevertheless, it is possible to make a continuous transfer, with a AXI4-S FIFO and a C program, by cleverly using the different registers.

## Maximum speed of a transfert

### AXI MM side, usage of the PS RAM

On the AXI MM side, the speed is limited by the PS : the AXI HP0/1/2/3 data width is limited to **128 bits**.
Considering that maximum clock frequency of the PS is **333 MHz**, the maximum transfert rate on the AXI MM side is **41.625 Gbps = 5.20 GB/s (or 5.20 Go/s)**.

> More precisely : 128 bits * 333 MHz = 42 624 x 10<sup>6</sup> bps = 41.625 Gbps = 5.20 GB/s (or 5.20 Go/s)


> With a more regular 300 MHz clock :  128 bits * 300 MHz = 38 400 x 10<sup>6</sup> bps = 37.5 Gbps = 4.6875 GB/s (or 4.6875 Go/s)

Moreover, HP1 and HP2 seem to share the same 128 bits AXI MM interface  [UG1085, figure 1-1, page 29](https://www.xilinx.com/support/documentation/user_guides/ug1085-zynq-ultrascale-trm.pdf), so the speed will not be maximal if they are both used at the same time.

#### Using Samples per seconds

Later, we will use a DMA to send or receive data with the DACs and ADCs. In general, it is more convenient to speak in GSa/s (Samples per seconds, or GSPS in Xilinx nomenclature) instead of GB/S (or Go/s). 

As we will see, DACs and ADCs use data coded on 16 bits, so **1 sample = 16 bits** = 2 Bytes (or 2 octets).

The maximum transfer rate on the AXI MM side is also **2.60 GSPS**.

> More precisely : 128 bits * 333 MHz = 42 624 x 10<sup>6</sup> bps = 41.625 Gbps = 2.60 GSPS


> With a more regular 300 MHz clock :  128 bits * 300 MHz = 38 400 x 10<sup>6</sup> bps = 37.5 Gbps = 2.34375 GSPS


### AXI MM side, usage of the PL RAM

As we have seen above, it is also possible to use the 4 GB of DDR4 SDRAM. In this case, the data width is limited to **512 bits**. 
Moreover, the DDR4 SDRAM memory need a clock frequency of **300 MHz**. In this case, the maximum transfert rate on the AXI MM side is **150 Gbps = 18.75 GB/s (or 18.75 Go/s)**. 

> More precisely : 512 bits * 300 MHz = 153 600 x 10<sup>6</sup> bps = 150 Gbps = 18.75 GB/s (or 18.75 Go/s)

#### Using Samples per seconds

The maximum transfer rate on the AXI MM side is also **9.375 GSPS**.

> More precisely : 512 bits * 300 MHz = 153 600 x 10<sup>6</sup> bps = 150 Gbps = 9.375 GSPS

### AXI S side

On the AXI S side, the speed is limited by the Intellectual Property (IP) that you use : generally, it is not possible to choose the parameters, except if you develop your own IP. 

## Maximum amount of RAM

It is possible to change the SODIMM RAM, in order to have up to **32 GB (or 32 Go) of memory in the PS**.


But unfortunately, it is not possible to increase the DDR4 SDRAM, because this memory is soldered, there is only **4 GB (or 4 Go) of memory in the PL**.


It is necessary to choose between speed and amount of memory : *32 GB @ 4.96 GB/s vs 4 GB @ 17.88 GB/s* !


# Use a FIFO

## Definition
In order to realize continuous data transfers, necessary to use DACs and ADCs later, we must make sure that the amount of upstream data is always at least superior to the amount of downstream data : that's why we will add a buffer (a FIFO), which will be clocked by 2 independent clocks, one for the input data and one for the output data. As the clocks are not the same, it is necessary to have a buffer to store the excess data.

A [FIFO](https://en.wikipedia.org/wiki/FIFO_(computing_and_electronics)) (first in, first out : the first in is the first out) is a method for organising the manipulation of a data buffer, where the oldest (first) entry, is processed first. 

## Size

A FIFO has a maximum memory depth of 32 768 (this is a hardware limit of the Xilinx IP). 

For an 128 bits AXI-S data width, this corresponds to **512 KB (or 512 Ko)**.

> More precisely : 32 768 * 128 bits = 4 194 304 bits = 4 Mb = **512 KB (or 512 Ko)**
