# Le protocole AXI4-Stream

Le protocole [AXI4-Stream](https://wiki.electroniciens.cnrs.fr/index.php/FPGA_CPLD_:_Guides_:_AXI4-Stream) est un protocole avec ``handshake'' qui utilise entre 2 et 9 signaux pour communiquer, bien que 3 à 4 signaux soient généralement suffisants et nécessaires.
Il s'agit d'un protocole ``maître-esclave'' : le maître envoie les données et l'esclave les reçoit.

![AXI4-Stream](./../images/AXI4-Stream.png?raw=true "AXI4-Stream Schema")

On a les 4 signaux usuels suivants :

- *TDATA* : vecteur de données
- *TVALID* : indique qu'une donnée est présente et valide pour l'esclave
- *TREADY* : présenté par l'esclave pour indiquer qu'il est prêt à recevoir les données
- *TLAST* : indique la fin d'une trame, utile lorsque les trames sont de taille variable

Les signaux *TDATA* et *TVALID* sont obligatoires pour réaliser une transaction AXI4-Stream.
Il est possible de se passer du signal *TREADY* et dans ce cas, la transmission est considérée comme ayant lieu sans confirmation de la part de l'esclave.
Une trame étant définie comme l'ensemble des données envoyées par le maître vers l'esclave, en une seule itération.
Enfin, afin d'envoyer plusieurs trames lors d'un même transfert, on utilise en plus le signal *TLAST*.

Le canal de lecture (**lecture des données depuis la RAM** et écriture dans le périphérique) de la DMA utilise les 3 premiers signaux pour envoyer le signal au DAC, alors que le canal d'écriture (lecture des données depuis périphérique et **ecriture dans la RAM**) impose en plus le signal *TLAST* pour recevoir les signaux de l'ADC.

Le nombre de trames souhaité est variable : en particulier si l'on veut avoir un signal continu !

# Convertisseurs RF

On retrouve différent convertisseurs RF (Radio-Frequence) que l'on souhaite exploiter sur la ZCU111.
Un [DAC](https://en.wikipedia.org/wiki/Digital-to-analog_converter) (Digital to Analog Converter) est un dispositif qui convertit un signal numérique en un signal analogique tandis qu'un [ADC](https://en.wikipedia.org/wiki/Analog-to-digital_converter) (Analog to Digital Converter) est un dispositif qui convertit un signal analogique en un signal numérique.

Comme la cible est un ``flux continu'' de données analogiques, il est logique d'utiliser le protocole AXI4-Stream afin d'envoyer aux DACs des données et d'en recevoir avec les ADCs.

On utilise l'IP [Zynq UltraScale+ RFSoC RF Data Converter](https://www.xilinx.com/products/intellectual-property/rf-data-converter.html) fournit par Xilinx afin de configurer les DACs et les ADCs et de les interfacer à la DMA au moyen du protocole AXI4-Stream.

# Caractéristiques techniques des convertisseurs RF

On retrouve les principales caractéristiques techniques de l'IP [Zynq UltraScale+ RFSoC RF Data Converter](https://www.xilinx.com/products/intellectual-property/rf-data-converter.html) fournit par Xilinx.

![RF_summary](./../images/RF_summary.png?raw=true "Zynq UltraScale+ RFSoC RF Data Converter Xilinx IP - Summary")

Les DACs et ADCs sont organisés en tuiles et en paires dans le SoC de la ZCU111 ([PG269](https://docs.xilinx.com/r/en-US/pg269-rf-data-converter)).
Chaque tuile partage la fréquence d'échantillonnage et l'utilisation (ou non) d'une PLL pour générer une horloge, ainsi que sa fréquence.

La carte additionnelle ''XM500 RFMC balun transformer add-on card`` propose différents connecteurs SMA pour exploiter facilement les différents convertisseurs.
La moitié des connecteurs exploitent des baluns, tandis que les autres sont de simples paires différentielles.
En particulier, certains connecteurs sont identifiés comme ''LF``(Low Frequency) puisqu'ils ont une bande analogique 0 - 1GHz, et d'autres comme ''HF`` (High Frequency) parce qu'ils ont une bande analogique 1 - 4GHz.

Il faut garder en tête le théorème d'échantillonnage de Nyquist-Shannon lorsque l'on utilise ces connecteurs : la fréquence d'échantillonage doit être au moins le double de la fréquence analogique désirée.
Avec une fréquence d'échantillonage de 2 GSa/s, on a une bande analogique maximale de 1 GHz et donc il faut utiliser les connecteurs LF, tandis qu'avec une fréquence d'échantillonage de 4 GSa/s, on a une bande analogique maximale de 2 GHz et donc il faut utiliser les connecteurs HF.
Dans notre cas, nous utilisons directement les paires différentielles, afin de conserver le signal original.

## Organisation des ADCs
