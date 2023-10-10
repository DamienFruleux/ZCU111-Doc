# Buffer memory

As we've just seen, the upstream part of the DMA uses the AXI4 Memory Map protocol, which is a packet-based protocol, while the downstream part uses the AXI4-Stream protocol, which is a stream-based protocol.
The 2 protocols are different, so it's the DMA that makes the connection.

It is nevertheless necessary to use a small buffer to temporarily store the data.

A FIFO ([First In, First Out](https://en.wikipedia.org/wiki/FIFO_(computing_and_electronics)) is a method for organizing the manipulation of a data buffer, where the oldest (first) entry, is processed first.

We use a FIFO with 2 independent clocks to act as a buffer: it is filled at one frequency, then emptied at another.

This FIFO acts as a reservoir, allowing us to "synchronize" 2 interfaces with different clocks.

The IP [AXI Streaming FIFO](https://www.xilinx.com/products/intellectual-property/axi_fifo.html) supplied by Xilinx is used to synchronize transfers between the DMA and the external device (DAC or ADC).

![AWG_FIFO](./../images/AWG_FIFO.png?raw=true "Disposition de la FIFO dans l'architecture de l'AWG")
![DGTZ_FIFO](./../images/DGTZ_FIFO.png?raw=true "Disposition de la FIFO dans l'architecture du DGTZ")

# Technical Characteristics of FIFO

These are the main technical features of the IP [AXI Streaming FIFO](https://www.xilinx.com/products/intellectual-property/axi_fifo.html) supplied by Xilinx.

![FIFO](./images/FIFO.png?raw=true "AXI Streaming FIFO Xilinx IP")

## FIFO size

The FIFO has a maximum memory depth of 32 768 (this is a hardware limit of the Xilinx IP) ([PG085](https://docs.xilinx.com/r/en-US/pg085-axi4stream-infrastructure)).

For a 256-bit AXI4-Stream data width (used by DACs), this corresponds to **1 MB of data, or 512 KSa**.

> More precisely: 32,768 * 256 bits = 8,388,608 bits = 8 Mb = 1 MB

For an AXI4-Stream data width of 128 bits (used by ADCs), this corresponds to **512 KB of data, or 256 KSa**.

> More precisely: 32,768 * 128 bits = 4,194,304 bits = 4 Mb = 512 KB

Note that the AXI4-Stream protocol provides additional bits to ensure smooth data transfer.
In particular, there is a _TLAST_ signal, which will be described in more detail later.
When we talk about throughput, we're talking only about useful data bits.








### 2-Independent Clock

Be sure to choose this option to be able to read and write in the FIFO with 2 different frequencies.


### 3-Continuous transfert

Thanks to this FIFO, which acts as a buffer, we also have the possibility of making continuous transfers. 

In fact, if the memory depth is sufficient, it is possible to launch a DMA transfer and before all the data is transferred to the AXI4-Stream side, it is possible to restart a DMA transfer and so on. 

In this way, the data is transferred continuously over time, without any "gaps" in the signal. 

It is therefore possible to artificially increase the size of a DMA transfer to exceed the limit of 67 108 863 bytes (or 64MB - 1).
