You can found a python exemple (with Pynq) [here](https://pynq.readthedocs.io/en/latest/pynq_libraries/dma.html), or a C exemple [here](https://lauri.xn--vsandi-pxa.com/hdl/zynq/xilinx-dma.html).

Pynq uses dynamic memory allocation, which means that it is not possible to use all available memory, let alone use it continuously.

Note that if it is possible to make continuous transfers in Python as in C, Pynq does not yet offer this method directly (to my knowledge). 
