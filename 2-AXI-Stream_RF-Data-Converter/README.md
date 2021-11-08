# Definition

Literally, a [Digital to Analog Converter](https://en.wikipedia.org/wiki/Digital-to-analog_converter) (DAC) is a system that converts a digital signal into an analog signal and an [Analog to Digital Converter](https://en.wikipedia.org/wiki/Analog-to-digital_converter) (ADC) is a system that converts an analog signal into a digital signal.

At xilinx, these peripherals **use the AXI S protocol as a basis**. Indeed, it is necessary to know that the IPs of the DACs and ADCs of Xilinx do not respect perfectly the AXI S protocol. In fact, we have continuous data flows, which use the AXI S protocol only as a basis. That is to say that the 2 converters send and receive data all the time on the *TDATA* signal, whatever the value of *TVALID* (and *TREADY*). It is therefore necessary to **condition these flows** so that they respect the AXI S protocol.

# AXIS Protocol

The [AXI4-Stream](https://wiki.electroniciens.cnrs.fr/index.php/FPGA_CPLD_:_Guides_:_AXI4-Stream) protocol is a handshake protocol that uses between 2 and 9 signals to communicate, although generally 3 to 4 are sufficient and necessary. It is a "master-slave" protocol: the master sends the data and the slave receives it. 

The following two signals are mandatory: 

- *TDATA* : data vector 
- *TVALID* : indicates that a data is present and valid for the slave

In most cases, an additional signal will be used: 

- *TREADY* : presented by the slave to indicate that it is ready to receive the data.

It is possible to dispense with the TREADY signal and in this case, transmission is considered to take place without confirmation from the slave. The following signal is also commonly used: 

- *TLAST* : indicates the end of a frame, useful when the frames are of variable size

A frame is defined as all the data sent from the master to the slave, in one iteration. It is therefore possible to send several frames in the same transfer. It is then necessary to adjust the TLAST signal.

The Read Channel of the DMA uses the first 3 signals to send the signal to the DACs, while the Write Channel imposes in addition the TLAST signal to receive the signals from the ADCs. The number of frames desired is variable: in particular if you want to have a continuous signal! 

# Signal Processing

It is important to know that the Xilinx DAC and ADC IPs do not perfectly respect the AXI S protocol. In fact, we have continuous data flows, which use the AXI
S protocol as a base only. That is to say that the 2 converters send and receive data all the time on the TDATA signal, whatever the value of TVALID (and TREADY). It is therefore necessary to condition these flows so that they respect the AXI S protocol.


Finally, the Read Channel of the DMA uses the first 3 signals to send the signal to the DACs,
while the Write Channel imposes in addition the TLAST signal to receive the signals from the ADCs.

## DAC
### Data rates
In practice, to operate at 2.048 Gsps, the DAC uses a flow of 256 bits: in 1 frame, there are 16 samples of 16 bits of data (i.e. 2 bytes). Clocked at 128 MHz, this gives a throughput of 4 GB/s (= 4Go/s). 

> More precisely: 256 bits * 128 MHz = 32 768 x 10^6 bps = 32 Gbps = 4 GB/s (= 4Go/s)

This is compatible with the maximum throughput proposed by the PS and PL.


### Operation
As the DAC does not take into account the TVALID signal, it permanently samples the data coming from TDATA. These being by definition not reset to 0 at the end of the transfer, the DAC continues to sample the last frame it reads in the previous IP! 

***It is then necessary to create an IP which imposes to put at zero the TDATA signal for the DAC when the TVALID signal coming from the DMA is 0 : I called this IP "DAC_sync"***

## ADC
### Data rates
In practice, to operate at 2.048 Gsps, the DAC uses a 128 bits flow: in 1 frame, there are 8 samples of 16 bits of data. Clocked at 256 MHz, this gives a throughput of 4 GB/s (= 4Go/s). 

> More precisely: 128 bits * 256 MHz = 32 768 x 10^6 bps = 32 Gbps = 4 GB/s (= 4Go/s)

This is compatible with the maximum throughput proposed by the PS and PL.
### Operation
Since the ADC continuously generates data on the TDATA signal, it never changes the TVALID signal, which is the TVALID signal, which remains at 1 permanently.

***It is then necessary to create an IP which will generate data frames only when the DMA is ready (TREADY signal) : I called this IP "ADC_sync".*** In fact, if a TDATA signal arrives on the DMA before it is initialized, it tends to block it.*


In addition, the DMA requires a TLAST signal which must also be generated according to the number of frames that we wish to acquire.

# Technical Characteristics of RF parts

## DACs

On ZCU111, the 8 DACs are arranged in 2 tiles (*228-229*) of 4 DACs each, organized in pairs (*0-1* and *2-3*)

On the XM500 board, the **pair 2-3 of tile 229 goes through a 0-1GHz bandpass filter** (called LF_TX), **the pair 0-1 of tile 229 goes through a 1-5GHz bandpass filter** (called HF_TX) and the **pairs 2-3 and 0-1 of tile 228 are not filtered** (the differential pair is available, without baluns).

DACs can work up to **6.554 Gsps**, on **14 bits** (in practice **6.144 Gsps** is more practical because it is a multiple of 1.024).

For each sampling rate, you need to adapt the AXI4-Stream bus upstream. In practice, it is only necessary to adjust the frequency of the clock, because one stream corresponds to **16 samples of 16 bits = 256 bits**.

## ADCs

On ZCU111, the 8 ADCs are arranged in 4 tiles (*224-225-226-227*) of 2 ADCs (*0-1*) each

On the XM500 board, the pair **0-1 of tile 224 goes through a 0-1GHz bandpass filter** (called LF_RX), **the pair 0-1 of tile 225 goes through a 1-5GHz bandpass filter** (called HF_RX) and the **pairs 0-1 of tiles 226 and 227 are not filtered** (the differential pair is available, without baluns)

ADCs can work up to **4.096 Gsps**, on **12 bits**.

For each sampling rate, you need to adapt the AXI4-Stream bus upstream. In practice, it is only necessary to adjust the frequency of the clock, because one stream corresponds to **8 samples of 16 bits = 128 bits**.



# Examples

Because of the large number of settings, I will give here 2 examples: 

  * 2 DACs (DAC229 Tile 1 Channel 2 & 3) @ 0-1 GHz (LF baluns) @ 1.024 Gsps -> "AWG_LF"

  * 2 DACs (DAC229 Tile 1 Channel 0 & 1) @ 1-5 GHz (HF baluns) @ 4.096 Gsps -> "AWG_HF" (later)
  
  * 2 ADCs (ADC224 Tile 0 Channel 0 & 1) @ 0-1 GHz (LF baluns) @ 1.024 Gsps -> "DIGITIZER_LF" (later)

  * 2 ADCs (ADC225 Tile 1 Channel 0 & 1) @ 1-5 GHz (HF baluns) @ 4.096 Gsps -> "DIGITIZER_HF" (later)

> In each case, DACs and ADCs are in the same pair (same clocks, sampling rate...)

> As I need the 2 DACs (or the 2 ADCs) to be perfectly synchronized, I send the 2 signals alternately in the same data stream: I therefore need an IP that separates the even data from the odd data.

# Data Interface & Clock Settings
# Bitrate, Frequency Sampling & Bandpass

