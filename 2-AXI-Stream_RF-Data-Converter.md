# Definitions of DAC and ADC

Literally, a [Digital to Analog Converter](https://en.wikipedia.org/wiki/Digital-to-analog_converter) (DAC) is a system that converts a digital signal into an analog signal and an [Analog to Digital Converter](https://en.wikipedia.org/wiki/Analog-to-digital_converter) (ADC) is a system that converts an analog signal into a digital signal.

At Xilinx, we use the [Zynq UltraScale+ RFSoC RF Data Converter](https://www.xilinx.com/products/intellectual-property/rf-data-converter.html) IP to **transfer data from the DMA to the DAC (or from the ADC to the DMA)** using an **AXI4-Stream interface**.

# AXI-Stream Protocol

The [AXI4-Stream](https://wiki.electroniciens.cnrs.fr/index.php/FPGA_CPLD_:_Guides_:_AXI4-Stream) protocol is a handshake protocol that uses between 2 and 9 signals to communicate, although generally 3 to 4 are sufficient and necessary. It is a "master-slave" protocol: the master sends the data and the slave receives it. 

The following two signals are mandatory: 

- *TDATA* : data vector 
- *TVALID* : indicates that a data is present and valid for the slave

In most cases, an additional signal will be used: 

- *TREADY* : presented by the slave to indicate that it is ready to receive the data.

It is possible to dispense with the TREADY signal and in this case, transmission is considered to take place without confirmation from the slave. The following signal is also commonly used: 

- *TLAST* : indicates the end of a frame, useful when the frames are of variable size

A frame is defined as all the data sent from the master to the slave, in one iteration. It is therefore possible to send several frames in the same transfer. It is then necessary to adjust the *TLAST* signal.

The Read Channel of the DMA uses the first 3 signals to send the signal to the DAC, while the Write Channel imposes in addition the *TLAST* signal to receive the signals from the ADC. 

The number of frames desired is variable: in particular if you want to have a continuous signal! (TODO)

# Signal Processing

It is important to know that the RF Data Converter IP do not perfectly respect the AXI4-Stream protocol. In fact, we have continuous data flows, which use the AXI4-Stream protocol only as a basis. That is to say that the 2 converters send and receive data all the time on the *TDATA* signal, whatever the value of *TVALID* (and *TREADY*). It is therefore necessary to **condition these flows** so that they respect the AXI4-Steam protocol.

## Using DAC

As the DAC does not take into account the *TVALID* signal, it permanently samples the data coming from *TDATA*. These being by definition not reset to 0 at the end of the transfer, the DAC continues to sample the last frame it reads in the previous IP (AXI4 FIFO) ! 

**It is then necessary to create an IP which imposes to put at zero the *TDATA* signal for the DAC when the *TVALID* signal coming from the DMA is 0 : I called this IP "FIFO_DAC_sync".**


## Using ADC

Since the ADC continuously generates data on the *TDATA* signal, it never changes the *TVALID* signal, which is the *TVALID* signal, which remains at 1 permanently.

**It is then necessary to create an IP which will generate data frames only when the DMA is ready (TREADY signal) : I called this IP "FIFO_ADC_sync".**


Moreover, as I wrote above, if a *TDATA* signal arrives on the DMA before it is initialized, it tends to block it. In addition, the DMA requires a *TLAST* signal which must also be generated according to the number of frames that we wish to acquire.

# Sampling Rate & Signal Duration

## Using DAC

For each sampling rate, you need to adapt the AXI4-Stream bus upstream. In practice, it is only necessary to adjust the frequency of the clock, because DACs always used a **flow of 256 bits**: in **1 frame**, there are **16 samples of 16 bits** of data (i.e. 16 samples of 2 bytes). So **it used a 256 bits AXI4-Stream interface**.

The ZCU111 has 8 DACs operating at **14-bit resolution**, up to a sampling rate of **6.554 GSa/s**, but **6.144 GSa/s** is more practical because it is a multiple of 256 (bits).

As we have already seen in the previous document (on DMA), we can generate a signal up to a sampling rate of 2 GSa/s with the PS (2.6 GSa/s max) and up to a sampling rate of 6 GSa/s with the PL (9.375 GSa/s max). I also remind that we can store 16 GSa in the PS and 2 GSa in the PL. 

I add below the details of the calculations, as well as the frequency of the necessary clock :

- For a sampling rate of **1.024 GSa/s**, we need a throughput of **2 GB/s**, and therefore a **64 MHz clock**. We get a signal of **16 secs for the PS** and **2 secs for the PL**.

> More precisely: 256 bits * 64 MHz = 16 384 x 10^6 bps = 16 Gbps = 2 GB/s (= 2 Go/s) = 1.024 GSa/s

- For a sampling rate of **2.048 GSa/s**, we need a throughput of **4 GB/s**, and therefore a **128 MHz clock**. We get a signal of **8 secs for the PS** and **1 sec for the PL**.

> More precisely: 256 bits * 128 MHz = 32 768 x 10^6 bps = 32 Gbps = 4 GB/s (= 4 Go/s) = 2.048 GSa/s

- For a sampling rate of **4.096 GSa/s**, we need a throughput of **8 GB/s**, and therefore a **256 MHz clock**. We get a signal of **0.5 secs for the PL** (PS is not fast enough).

> More precisely: 256 bits * 256 MHz = 65 536 x 10^6 bps = 64 Gbps = 8 GB/s (= 8 Go/s) = 4.096 GSa/s

- For a sampling rate of **6.144 GSa/s**, we need a throughput of **12 GB/s**, and therefore a **384 MHz clock**. We get a signal of **0.333 secs for the PL** (PS is not fast enough).

> More precisely: 256 bits * 384 MHz = 98 304 x 10^6 bps = 96 Gbps = 12 GB/s (= 12 Go/s) = 6.144 GSa/s

## Using ADC

For each sampling rate, you need to adapt the AXI4-Stream bus upstream. In practice, it is only necessary to adjust the frequency of the clock, because ADCs always used a **flow of 128 bits**: in **1 frame**, there are **8 samples of 16 bits** of data (i.e. 8 samples of 2 bytes). So **it used a 128 bits AXI4-Stream interface**.

The ZCU111 has 8 DACs operating at **12-bit resolution**, up to a sampling rate of **4.096 GSa/s**.

As we have already seen in the previous document (on DMA), we can acquire a signal up to a sampling rate of 2 GSa/s with the PS (2.6 GSa/s max) and up to a sampling rate of 6 GSa/s with the PL (9.375 GSa/s max). I also remind that we can store 16 GSa in the PS and 2 GSa in the PL. 

I add below the details of the calculations, as well as the frequency of the necessary clock :

- For a sampling rate of **1.024 GSa/s**, we need a throughput of **2 GB/s**, and therefore a **128 MHz clock**. We get a signal of **16 secs for the PS** and **2 secs for the PL**.

> More precisely: 128 bits * 128 MHz = 16 384 x 10^6 bps = 16 Gbps = 2 GB/s (= 2 Go/s) = 1.024 GSa/s

- For a sampling rate of **2.048 GSa/s**, we need a throughput of **4 GB/s**, and therefore a **256 MHz clock**. We get a signal of **8 secs for the PS** and **1 sec for the PL**.

> More precisely: 128 bits * 256 MHz = 32 768 x 10^6 bps = 32 Gbps = 4 GB/s (= 4 Go/s) = 2.048 GSa/s

- For a sampling rate of **4.096 GSa/s**, we need a throughput of **8 GB/s**, and therefore a **512 MHz clock**. We get a signal of **0.5 secs for the PL** (PS is not fast enough).

> More precisely: 128 bits * 512 MHz = 65 536 x 10^6 bps = 64 Gbps = 8 GB/s (= 8 Go/s) = 4.096 GSa/s

## Use multiple channels at the same time



### Use multi DMA individualy



### Use multi DMA together



### Use sigle DMA

TODO

> As I need the 2 DACs (or the 2 ADCs) to be perfectly synchronized, I send the 2 signals alternately in the same data stream: I therefore need an IP that separates the even data from the odd data.

### Compresse datas

# Technical Characteristics of RF parts

On ZCU111, DACs and ADCs are organized in tiles and pairs. Each tile shares the sampling frequency and the use (or not) of a PLL to generate a clock, and its frequency.

Note that XM500 RFMC balun transformer add-on card provides different SMA connectors to operate DACs and ADCs.
Half of the ports use baluns and RF filters, while the others are differential pairs. 
In particular, some ports are identified as LF, because they have a 0-1GHz low-cut filter, and others as HF, because they have a 1-4GHz bandpass filter.

We must be careful with the sampling frequency/filter pairs in order to respect the Nyquist-Shannon sampling theorem.
If the sampling frequency is 1 GSa/s, the analog signal cannot exceed the frequency of 512 MHz, for 2 GSa/s it is 1GHz, for 4 GSa/s it is 2 GHz and for 6 GSa it is 3GHz.

## DACs organization

On ZCU111, the 8 DACs are arranged in 2 tiles (*228-229*) of 4 DACs each, organized in pairs (*0,1* and *2,3*).

On the XM500 board, **the pair 2,3 of tile 229 goes through a 0-1GHz bandpass filter** (called LF_TX), **the pair 0,1 of tile 229 goes through a 1-4GHz bandpass filter** (called HF_TX) and **the pairs 2,3 and 0,1 of tile 228 are not filtered** (the differential pair is available, without baluns).

In conclusion, we have : 

- DAC Tile 228 - DAC Pair 0,1 - not filtered (differential pair) - SMA J26/J27 and J20/J21
- DAC Tile  228 - DAC Pair 2,3 - not filtered (differential pair) - SMA J22/J23 and J24/J25
- DAC Tile  229 - DAC Pair 0,1 - 1-4 GHz (HF_TX) - SMA J7 and J8
- DAC Tile  229 - DAC Pair 2,3 - 0-1 GHz (LF_TX) - SMA J6 and J5

## ADCs organization

On ZCU111, the 8 ADCs are arranged in 4 tiles (*224-225-226-227*) of 2 ADCs each, organized in pair (*0-1*).

On the XM500 board, the pair **0-1 of tile 224 goes through a 0-1GHz bandpass filter** (called LF_RX), **the pair 0-1 of tile 225 goes through a 1-4GHz bandpass filter** (called HF_RX) and the **pairs 0-1 of tiles 226 and 227 are not filtered** (the differential pair is available, without baluns)

In conclusion, we have : 

- DAC Tile 227 - DAC Pair 0,1 - not filtered (differential pair) - SMA J32/J33 and J34/J35
- DAC Tile 226 - DAC Pair 0,1 - not filtered (differential pair) - SMA J36/J37 and J39/J40
- DAC Tile  225 - DAC Pair 0,1 - 1-4 GHz (HF_RX) - SMA J2 and J1
- DAC Tile  224 - DAC Pair 0,1 - 0-1 GHz (LF_RX) - SMA J4 and J3


TODO Example
> In each case, DACs and ADCs are in the same pair (same clocks, sampling rate...)