# Mémoire tampon

Comme nous venons de le voir, dans le cas de l'AWG, la partie en amont de la DMA utilise le protocole AXI4 Memory Map, qui est un protocole qui fonctionne par paquets, alors que la partie en aval utilise le protocole AXI4-Stream, qui est un protocole qui fonctionne par flux de données.
Dans le cadre du DGTZ, le chemin est inversé.
Les 2 protocoles sont différents et c'est donc la DMA qui fait la liaison.

Il est néanmoins nécessaire d'utiliser une petite mémoire tampon afin de stocker temporairement les données.

Une FIFO ([First In, First Out](https://en.wikipedia.org/wiki/FIFO_(computing_and_electronics))) est une méthode d'organisation de la gestion d'une mémoire tampon, où l'entrée la plus ancienne est la première traitée.

On utilise une FIFO avec 2 horloges indépendantes qui nous permet de jouer le rôle de mémoire tampon : elle est remplie à une fréquence, puis vidée à une autre.

Cette FIFO joue donc le rôle de réservoir permettant de "synchroniser" 2 interfaces ayant des horloges différentes.

On utilise pour cela l'IP [AXI Streaming FIFO](https://www.xilinx.com/products/intellectual-property/axi_fifo.html) fournit par Xilinx afin de synchroniser les transferts entre la DMA et le périphérique externe (DAC ou ADC)

![AWG_FIFO](./../images/AWG_fifo.png?raw=true "Disposition de la FIFO dans l'architecture de l'AWG")
![DGTZ_FIFO](./../images/DGTZ_fifo.png?raw=true "Disposition de la FIFO dans l'architecture du DGTZ")

# Caractéristiques techniques de la FIFO

On retrouve les principales caractéristiques techniques de l'IP [AXI Streaming FIFO](https://www.xilinx.com/products/intellectual-property/axi_fifo.html) fournit par Xilinx.

![FIFO](./../images/FIFO.png?raw=true "IP Xilinx AXI Streaming FIFO")

## Taille de la FIFO

La FIFO a une profondeur mémoire maximale de 32 768 (il s'agit d'une limite matérielle de l'IP de Xilinx) ([PG085](https://docs.xilinx.com/r/en-US/pg085-axi4stream-infrastructure)).

Les DACs utilisent une largeur de données AXI4-Stream de 256 bits (16 échantillons de 16 bits), cela correspond à **1 Mo de données, soit 512 KSa**.

> Plus précisément : 32 768 * 256 bits = 8 388 608 bits = 8 Mb = 1 Mo

Les ADCs utilisent une largeur de données AXI4-Stream de 128 bits (8 échantillons de 16 bits), cela correspond à **512 Ko de données, soit 256 KSa**.

> Plus précisément : 32 768 * 128 bits = 4 194 304 bits = 4 Mb = 512 Ko

On remarque que le protocole AXI4-Stream propose des bits supplémentaires afin d'assurer le bon déroulé du transfert.
En particulier, on retrouve un signal _TLAST_, qui sera détaillé par la suite.
Quand on parle de débit, on évoque seulement les bits de données utiles.

## Horloges indépendantes

Il faut choisir cette option pour pouvoir lire et écrire dans la FIFO à deux fréquences différentes.

