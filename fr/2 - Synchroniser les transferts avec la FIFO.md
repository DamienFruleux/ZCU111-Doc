# Mémoire tampon

Comme nous venons de le voir, la partie en amont de la DMA utilise le protocole AXI4 Memory Map, qui est un protocole qui fonctionne par paquets, alors que la partie en aval utilise le protocole AXI4-Stream, qui est un protocole qui fonctionne par flux de données.
Les 2 protocoles sont différents et c'est donc la DMA qui fait la liaison.

Il est néanmoins nécessaire d'utiliser une petite mémoire tampon afin de stocker temporairement les données.

Une FIFO ([First In, First Out](https://en.wikipedia.org/wiki/FIFO_(computing_and_electronics))) est une méthode d'organisation de la gestion d'une mémoire tampon, où l'entrée la plus ancienne est la première traitée.

On utilise une FIFO avec 2 horloges indépendantes qui nous permet de jouer le rôle de mémoire tampon : elle est remplie à une fréquence, puis vidée à une autre.

Cette FIFO joue donc le rôle de réservoir permettant de "synchroniser" 2 interfaces ayant des horloges différentes.

On utilise pour cela l'IP [AXI Streaming FIFO](https://www.xilinx.com/products/intellectual-property/axi_fifo.html) fournit par Xilinx afin de synchroniser les transferts entre la DMA et le périphérique externe (DAC ou ADC)

![AWG_FIFO](./../images/AWG_FIFO.png?raw=true "Disposition de la FIFO dans l'architecture de l'AWG")
![DGTZ_FIFO](./../images/DGTZ_FIFO.png?raw=true "Disposition de la FIFO dans l'architecture du DGTZ")

# Caractéristiques techniques de la FIFO

On retrouve les principales caractéristiques techniques de l'IP [AXI Streaming FIFO](https://www.xilinx.com/products/intellectual-property/axi_fifo.html) fournit par Xilinx.

![FIFO](./../images/FIFO.png?raw=true "IP Xilinx AXI Streaming FIFO")

## Taille de la FIFO

La FIFO a une profondeur mémoire maximale de 32 768 (il s'agit d'une limite matérielle de l'IP de Xilinx) ([PG085](https://docs.xilinx.com/r/en-US/pg085-axi4stream-infrastructure)).

Pour une largeur de données AXI4-Stream de 256 bits (qu'utilisent les DACs), cela correspond à **1 Mo de données, soit 512 KSa**.

> Plus précisément : 32 768 * 256 bits = 8 388 608 bits = 8 Mb = 1 Mo

Pour une largeur de données AXI4-Stream de 128 bits (qu'utilisent les ADCs), cela correspond à **512 Ko de données, soit 256 KSa**.

> Plus précisément : 32 768 * 128 bits = 4 194 304 bits = 4 Mb = 512 Ko

On remarque que le protocole AXI4-Stream propose des bits supplémentaires afin d'assurer le bon déroulé du transfert.
En particulier, on retrouve un signal _TLAST_, qui sera détaillé par la suite.
Quand on parle de débit, on évoque seulement les bits de données utiles.








### Independent Clock

Be sure to choose this option to be able to read and write in the FIFO with 2 different frequencies.


### Continuous transfert

Thanks to this FIFO, which acts as a buffer, we also have the possibility of making continuous transfers. 

In fact, if the memory depth is sufficient, it is possible to launch a DMA transfer and before all the data is transferred to the AXI4-Stream side, it is possible to restart a DMA transfer and so on. 

In this way, the data is transferred continuously over time, without any "gaps" in the signal. 

It is therefore possible to artificially increase the size of a DMA transfer to exceed the limit of 67 108 863 bytes (or 64MB - 1).
