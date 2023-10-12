# AXI4-Stream protocol

The [AXI4-Stream] protocol (https://wiki.electroniciens.cnrs.fr/index.php/FPGA_CPLD_:_Guides_:_AXI4-Stream) is a "handshake" protocol that uses between 2 and 9 signals to communicate, although 3 to 4 signals are generally sufficient and necessary.
It is a master-slave protocol: the master sends data and the slave receives it.

![AXI4-Stream](./../images/AXI4-Stream.png?raw=true "AXI4-Stream Schema")

We have the following 4 common signals:

- *TDATA*: data vector
- *TVALID*: indicates that data is present and valid for the slave
- *TREADY*: presented by the slave to indicate that it is ready to receive data
- *TLAST*: indicates the end of a frame, useful when frames are of variable size

The *TDATA* and *TVALID* signals are mandatory to complete an AXI4-Stream transaction.
It is possible to dispense with the *TREADY* signal, in which case transmission is considered to have taken place without confirmation from the slave.
A frame is defined as all the data sent by the master to the slave, in a single iteration.
Finally, the *TLAST* signal is also used to send multiple frames in a single transfer.

The DMA read channel (**read data from RAM** and write it to device) uses the first 3 signals to send the signal to the DAC, while the write channel (read data from device and **write it to RAM**) also requires the *TLAST* signal to receive signals from the ADC.

The number of frames required is variable: especially if you want a continuous signal!

# RF converters

There are various RF (Radio-Frequency) converters to be used on the ZCU111.
A [DAC](https://en.wikipedia.org/wiki/Digital-to-analog_converter) (Digital to Analog Converter) is a device that converts a digital signal into an analog signal, while an [ADC](https://en.wikipedia.org/wiki/Analog-to-digital_converter) (Analog to Digital Converter) is a device that converts an analog signal into a digital signal.

Since the target is a "continuous flow" of analog data, it makes sense to use the AXI4-Stream protocol to send data to the DACs and receive data from the ADCs.

The IP [Zynq UltraScale+ RFSoC RF Data Converter](https://www.xilinx.com/products/intellectual-property/rf-data-converter.html) supplied by Xilinx is used to configure the DACs and ADCs and interface them to the DMA using the AXI4-Stream protocol.

# Technical specifications of RF converters

These are the main technical specifications of the IP [Zynq UltraScale+ RFSoC RF Data Converter](https://www.xilinx.com/products/intellectual-property/rf-data-converter.html) supplied by Xilinx.

![RF_summary](./../images/RF_summary.png?raw=true "Zynq UltraScale+ RFSoC RF Data Converter Xilinx IP - Summary")

DACs and ADCs are organized in tiles and pairs in the ZCU111 SoC ([PG269](https://docs.xilinx.com/r/en-US/pg269-rf-data-converter)).
Each tile shares the sampling frequency and the use (or not) of a PLL to generate a clock, as well as its frequency.

The ''XM500 RFMC balun transformer add-on card'' features various SMA connectors for easy operation of the various converters.
Half the connectors use baluns, while the others are simple differential pairs.
In particular, some connectors are identified as ''LF'' (Low Frequency) because they have a 0 - 1GHz analog band, and others as ''HF'' (High Frequency) because they have a 1 - 4GHz analog band.

Keep in mind the Nyquist-Shannon sampling theorem when using these connectors: the sampling frequency must be at least twice the desired analog frequency.
With a sampling frequency of 2 GSa/s, we have a maximum analog bandwidth of 1 GHz, so we need to use LF connectors, while with a sampling frequency of 4 GSa/s, we have a maximum analog bandwidth of 2 GHz, so we need to use HF connectors.
In our case, we use the differential pairs directly, in order to preserve the original signal.


















## A- DACs organization

![RF_DAC_1](./images/RF_DAC_1.png?raw=true "Zynq UltraScale+ RFSoC RF Data Converter Xilinx IP - RF-DAC")
![RF_DAC_2](./images/RF_DAC_2.png?raw=true "Zynq UltraScale+ RFSoC RF Data Converter Xilinx IP - RF-DAC")

On ZCU111, the 8 DACs are arranged in 2 tiles (*228-229*) of 4 DACs each, organized in pairs (*0,1* and *2,3*).

On the XM500 board, **the pair 2,3 of tile 229 goes through a 0-1GHz bandpass filter** (called LF_TX), **the pair 0,1 of tile 229 goes through a 1-4GHz bandpass filter** (called HF_TX) and **the pairs 2,3 and 0,1 of tile 228 are not filtered** (the differential pair is available, without baluns).

In conclusion, we have : 

- DAC Tile 228 - DAC Pair 0,1 - not filtered (differential pair) - SMA J26/J27 and J20/J21
- DAC Tile  228 - DAC Pair 2,3 - not filtered (differential pair) - SMA J22/J23 and J24/J25
- DAC Tile  229 - DAC Pair 0,1 - 1-4 GHz (HF_TX) - SMA J7 and J8
- DAC Tile  229 - DAC Pair 2,3 - 0-1 GHz (LF_TX) - SMA J6 and J5

## B- ADCs organization

![RF_ADC_1](./images/RF_ADC_1.png?raw=true "Zynq UltraScale+ RFSoC RF Data Converter Xilinx IP - RF-ADC")
![RF_ADC_2](./images/RF_ADC_2.png?raw=true "Zynq UltraScale+ RFSoC RF Data Converter Xilinx IP - RF-ADC")

On ZCU111, the 8 ADCs are arranged in 4 tiles (*224-225-226-227*) of 2 ADCs each, organized in pair (*0-1*).

On the XM500 board, the pair **0-1 of tile 224 goes through a 0-1GHz bandpass filter** (called LF_RX), **the pair 0-1 of tile 225 goes through a 1-4GHz bandpass filter** (called HF_RX) and the **pairs 0-1 of tiles 226 and 227 are not filtered** (the differential pair is available, without baluns)

In conclusion, we have : 

- DAC Tile 227 - DAC Pair 0,1 - not filtered (differential pair) - SMA J32/J33 and J34/J35
- DAC Tile 226 - DAC Pair 0,1 - not filtered (differential pair) - SMA J36/J37 and J39/J40
- DAC Tile  225 - DAC Pair 0,1 - 1-4 GHz (HF_RX) - SMA J2 and J1
- DAC Tile  224 - DAC Pair 0,1 - 0-1 GHz (LF_RX) - SMA J4 and J3

## C- Sampling Rate, AXI4-Stream Clock & PLL

The converters in the same tile share different elements : 

- The sampling frequency
- The AXI4-Stream clock
- The PLL (optional) which allows to generate a clock at a desired frequency if needed

The selected sampling rate determines the frequency of the AXI4-Stream clock.

It is possible to generate a clock from this IP that can be used as an input (loop). It will be necessary to multiply the frequency of this clock by 2 when using ADCs (nothing to do for DACs).

We will keep the default value of the PLL (409.600 MHz) which is adapted to our case. However, it is possible to change it, in particular if we wish to use a sampling frequency that is not a multiple of 2, but a multiple of 10 (1.024 GSa/s vs 1 GSa/s). 

The converters in the same tile share the same clocks, so we can be sure that they will be perfectly synchronised with each other. Therefore, the converters must be correctly associated, giving preference to those in the same tile, as explained above.

However, it must be ensured that the digital signals from both channels are available at the same time to be converted together.












# 3- Signal Processing

It is important to know that the RF Data Converter IP do not perfectly respect the AXI4-Stream protocol. In fact, we have continuous data flows, which use the AXI4-Stream protocol only as a basis. That is to say that the 2 converters send and receive data all the time on the *TDATA* signal, whatever the value of *TVALID* (and *TREADY*). It is therefore necessary to **condition these flows** so that they respect the AXI4-Steam protocol and interact properly with .

## A- Using DAC

As the DAC does not take into account the *TVALID* signal, it permanently samples the data coming from *TDATA*. These being by definition not reset to 0 at the end of the transfer, the DAC continues to sample the last frame it reads in the previous IP (AXI4 FIFO) ! %TODO

**It is then necessary to create an IP which imposes to put at zero the *TDATA* signal for the DAC when the *TVALID* signal coming from the DMA is 0 : I called this IP "FIFO_sync".**

![PS-to-DAC](./images/PS-to-DAC.png?raw=true "PS-to-DAC Schema")

## B- Using ADC

Since the ADC continuously generates data on the *TDATA* signal, it never changes the *TVALID* signal, which is the *TVALID* signal, which remains at 1 permanently.

**It is then necessary to create an IP which will generate data frames only when the DMA is ready (TREADY signal) : I called this IP "FIFO_ADC_sync".**

Moreover, as I wrote above, if a *TDATA* signal arrives on the DMA before it is initialized, it tends to block it. In addition, the DMA requires a *TLAST* signal which must also be generated according to the number of frames that we wish to acquire.

todo : need to complet, still in progress

![ADC-to-PS_1](./images/ADC-to-PS_1.png?raw=true "ADC-to-PS Schema")
![ADC-to-PS_2](./images/ADC-to-PS_2.png?raw=true "ADC-to-PS Schema")












# 4- Sampling Rate & Signal Duration

## A- Using DAC

For each sampling rate, you need to adapt the AXI4-Stream bus upstream. In practice, it is only necessary to adjust the frequency of the clock, because DACs always used a **flow of 256 bits**: in **1 frame**, there are **16 samples of 16 bits** of data (i.e. 16 samples of 2 bytes). So **it used a 256 bits AXI4-Stream interface**.

The ZCU111 has 8 DACs operating at **14-bit resolution**, up to a sampling rate of **6.554 GSa/s**, but **6.144 GSa/s** is more practical because it is a multiple of 256 (bits). In practice, we will use in the code unsigned integers coded on 16 bits (uint16_t) : the 2 most significant bits are not representative.

As we have already seen previously, we can generate a signal up to a sampling rate of 2.048 GSa/s with the PS (2.6 GSa/s max) and up to a sampling rate of 6.144 GSa/s with the PL (9.375 GSa/s max). I also remind that we can store up to 14 GSa (and not 16 GSa, this is not a mistake, we will explain next how, with RAM upgrade) and up to 2 GSa in the PL. 

I add below the details of the calculations, as well as the frequency of the necessary clock :

- For a sampling rate of **1.024 GSa/s**, we need a throughput of **2 GB/s**, and therefore a **64 MHz clock**. We get a signal of **14 secs for the PS** and **2 secs for the PL**.

> More precisely: 256 bits * 64 MHz = 16 384 x 10^6 bps = 16 Gbps = 2 GB/s (= 2 Go/s) = 1.024 GSa/s

- For a sampling rate of **2.048 GSa/s**, we need a throughput of **4 GB/s**, and therefore a **128 MHz clock**. We get a signal of **7 secs for the PS** and **1 sec for the PL**.

> More precisely: 256 bits * 128 MHz = 32 768 x 10^6 bps = 32 Gbps = 4 GB/s (= 4 Go/s) = 2.048 GSa/s

- For a sampling rate of **4.096 GSa/s**, we need a throughput of **8 GB/s**, and therefore a **256 MHz clock**. We get a signal of **0.5 secs for the PL** (PS is not fast enough).

> More precisely: 256 bits * 256 MHz = 65 536 x 10^6 bps = 64 Gbps = 8 GB/s (= 8 Go/s) = 4.096 GSa/s

- For a sampling rate of **6.144 GSa/s**, we need a throughput of **12 GB/s**, and therefore a **384 MHz clock**. We get a signal of **0.333 secs for the PL** (PS is not fast enough).

> More precisely: 256 bits * 384 MHz = 98 304 x 10^6 bps = 96 Gbps = 12 GB/s (= 12 Go/s) = 6.144 GSa/s

## B- Using ADC

For each sampling rate, you need to adapt the AXI4-Stream bus upstream. In practice, it is only necessary to adjust the frequency of the clock, because ADCs always used a **flow of 128 bits**: in **1 frame**, there are **8 samples of 16 bits** of data (i.e. 8 samples of 2 bytes). So **it used a 128 bits AXI4-Stream interface**.

The ZCU111 has 8 DACs operating at **12-bit resolution**, up to a sampling rate of **4.096 GSa/s**. In practice, we will use in the code unsigned integers coded on 16 bits (uint16_t) : the 4 most significant bits are not representative. This still represents 25% of the data transferred unnecessarily. We will see later that it is possible to compress the data to avoid such waste

As we have already seen in the previous document (on DMA), we can acquire a signal up to a sampling rate of 2.048 GSa/s with the PS (2.6 GSa/s max) and up to a sampling rate of 4.096 GSa/s with the PL (9.375 GSa/s max). I also remind that we can store up to 14 GSa (and not 16 GSa, this is not a mistake, we will explain next how, with RAM upgrade) and up to 2 GSa in the PL. 

I add below the details of the calculations, as well as the frequency of the necessary clock :

- For a sampling rate of **1.024 GSa/s**, we need a throughput of **2 GB/s**, and therefore a **128 MHz clock**. We get a signal of **14 secs for the PS** and **2 secs for the PL**.

> More precisely: 128 bits * 128 MHz = 16 384 x 10^6 bps = 16 Gbps = 2 GB/s (= 2 Go/s) = 1.024 GSa/s

- For a sampling rate of **2.048 GSa/s**, we need a throughput of **4 GB/s**, and therefore a **256 MHz clock**. We get a signal of **7 secs for the PS** and **1 sec for the PL**.

> More precisely: 128 bits * 256 MHz = 32 768 x 10^6 bps = 32 Gbps = 4 GB/s (= 4 Go/s) = 2.048 GSa/s

- For a sampling rate of **4.096 GSa/s**, we need a throughput of **8 GB/s**, and therefore a **512 MHz clock**. We get a signal of **0.5 secs for the PL** (PS is not fast enough).

> More precisely: 128 bits * 512 MHz = 65 536 x 10^6 bps = 64 Gbps = 8 GB/s (= 8 Go/s) = 4.096 GSa/s






# 6- Clocks settings 

todo : PLL / LMK04208 & LMX2594 / DAC_CLKIN & ADC_CLKIN







# 7- Use multiple channels at the same time

It is often necessary to work with several converters at the same time.

The obvious method is easy to realize: just multiply the DMAs and run the control code in parallel.

## A- Use multi DMA 

However, most of the time it is necessary that the signals are synchronized, so that the information is available at the same time. 
This complicates things, because even if you write the code in C in a very optimized way, it is difficult to guarantee that the data is perfectly synchronized and available at exactly the same time. In some critical cases, this approximation is not possible, which is my case.

## B- Use single DMA

### 1-With DACs

For 2 perfectly synchronized DACs, the 2 signals (or channels) must be sent alternately in the same data stream. For this, we will use a little trick which consists in placing on the even elements of our signal the samples of the DAC_A and on the odd elements of our signal, the samples of the DAC_B.

If we keep the same sampling frequency of the **total** signal and the same number of **total** samples, this obviously leads to a division by 2 of the sampling frequency **per channel**, and a division by 2 of the number of samples **per channel**.

> For example, instead of having in the PS 1 signal composed of 14 GSa and sampled at 2.048 GSa/s, we will have 2 signals composed **in total** of 14 GSa and sampled **in total** at 2.048 GSa/s, which gives 7 GSa **per channel** and 1.024 GSa/s **per channel**.

In the code, it is necessary to send to the DMA a buffer which contains the samples of channel A on the even data and the samples of channel B on the odd data.

In the design, after the DMA, I need an IP that separates the even data from the odd data and sends them to channel A and channel B respectively : I called this IP "AXIS_SPLITTER_2".

However, it is necessary to double the width of the upstream buses if you want to keep the same sampling rate as before, so they go from 256 bits (for 1 DAC at 2,048 GSa/s) to 512 bits (for 2 DACs at 1,024 GSa/s). As a result, the required clock is reduced from 128MHz to 64MHz.

![AXIS_SPLITTER_2](./images/AXIS_SPLITTER_2.png?raw=true "AXIS_SPLITTER_2 IP")

### 2-With ADCs

For 2 perfectly synchronized ADCs, we do exactly the same thing, but in the opposite direction.

In the design, before the DMA, we need an IP that joins the samples of channel A and channel B alternately on the even and odd data of the data stream : I called this IP "AXIS_COMBINER_2".

In the same way, it is necessary to double the width of the upstream buses if you want to keep the same sampling rate as before, so they go from 128 bits (for 1 ADC at 2,048 GSa/s) to 256 bits (for 2 ADCs at 1,024 GSa/s). As a result, the required clock is reduced from 256MHz to 128MHz.

In the code, it is necessary to receive from the DMA the buffer that contains the samples of channel A on the even data and the samples of channel B on the odd data.

![AXIS_COMBINER_2](./images/AXIS_COMBINER_2.png?raw=true "AXIS_COMBINER_2 IP")












# 8- Compressing the data

It is possible to compress the data: in fact, we specified a little earlier that the DAC data is on 14 bits, and the ADC data on 12 bits.

However, the AXI4-Stream bus requires data on 16 bits (like the code afterwards which uses unsigned integers on 16 bits: uint16_t), the remaining 2 or 4 MSB bits are simply set to zero!

It is therefore possible to save respectively 12.5 and 25% of the flow necessary to send the same information. 
In the case of DACs, this is relatively low, but in the case of ADCs, the compression is very important!

Although this is not necessary in the examples presented for the moment, it will be useful if we wish to use the 4 SFP28 connectors later on. Indeed, with a data rate of 4 * 25Gbps = 100 Gbps, i.e. 12.5 GB/s (or 12.5 Go/s), it is not possible to send 4 data streams from the ADCs at 2 GSa/s each, because this requires a cumulative data rate of 4*2.048 GSa/s = 8.192 GSa/s = 16 GB/s (= 16 Go/s). With a 25% compression (12 bits instead of 16), we get a throughput of 12 GB/s (= 12 Go/s), which is well compatible with the throughput of the 4 SFP28 ports.

To do this, we need to perform an IP in Vivado that removes the 4 MSB bits from the RF signals of the ADCs and aggregates 4 * 12 bits into 3 * 16 bits before sending the data to the DMA. 

In the code, it will be necessary to add a function which selects the 12 bits of MSB and which adds 4 bits for each sample, in order to have again data on unsigned integers of 16 bits!

![DATA_COMPRESSION](./images/DATA_COMPRESSION.png?raw=true "DATA_COMPRESSION IP")

todo : add schema
