# Le protocole AXI4-Stream

Le protocole [AXI4-Stream](https://wiki.electroniciens.cnrs.fr/index.php/FPGA_CPLD_:_Guides_:_AXI4-Stream) est un protocole avec "handshake" qui utilise entre 2 et 9 signaux pour communiquer, bien que 3 à 4 signaux soient généralement suffisants et nécessaires.
Il s'agit d'un protocole "maître-esclave" : le maître envoie les données et l'esclave les reçoit.

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

Comme la cible est un "flux continu" de données analogiques, il est logique d'utiliser le protocole AXI4-Stream afin d'envoyer aux DACs des données et d'en recevoir avec les ADCs.

On utilise l'IP [Zynq UltraScale+ RFSoC RF Data Converter](https://www.xilinx.com/products/intellectual-property/rf-data-converter.html) fournit par Xilinx afin de configurer les DACs et les ADCs et de les interfacer à la DMA au moyen du protocole AXI4-Stream.

# Caractéristiques techniques des convertisseurs RF

On retrouve les principales caractéristiques techniques de l'IP [Zynq UltraScale+ RFSoC RF Data Converter](https://www.xilinx.com/products/intellectual-property/rf-data-converter.html) fournit par Xilinx.

![RF_summary](./../images/RF_summary.png?raw=true "Zynq UltraScale+ RFSoC RF Data Converter Xilinx IP - Summary")

Les DACs et ADCs sont organisés en tuiles et en paires dans le SoC de la ZCU111 ([PG269](https://docs.xilinx.com/r/en-US/pg269-rf-data-converter)).
Chaque tuile partage la fréquence d'échantillonnage et l'utilisation (ou non) d'une PLL pour générer une horloge, ainsi que sa fréquence.

La carte additionnelle "XM500 RFMC balun transformer add-on card" propose différents connecteurs SMA pour exploiter facilement les différents convertisseurs.
La moitié des connecteurs exploitent des baluns, tandis que les autres sont de simples paires différentielles.
En particulier, certains connecteurs sont identifiés comme "LF"(Low Frequency) puisqu'ils ont une bande analogique 0 - 1GHz, et d'autres comme "HF" (High Frequency) parce qu'ils ont une bande analogique 1 - 4GHz.

Il faut garder en tête le théorème d'échantillonnage de Nyquist-Shannon lorsque l'on utilise ces connecteurs : la fréquence d'échantillonage doit être au moins le double de la fréquence analogique désirée.
Avec une fréquence d'échantillonage de 2 GSa/s, on a une bande analogique maximale de 1 GHz et donc il faut utiliser les connecteurs LF, tandis qu'avec une fréquence d'échantillonage de 4 GSa/s, on a une bande analogique maximale de 2 GHz et donc il faut utiliser les connecteurs HF.
Dans notre cas, nous utilisons directement les paires différentielles, afin de conserver le signal original.

## Organisation des DACs

Sur la ZCU111, les 8 DACs sont disposés en 2 tuiles (228-229) de 4 DACs chacune, organisées par paires (0,1 et 2,3).

![RF_DAC](./../images/RF_DAC.png?raw=true "Zynq UltraScale+ RFSoC RF Data Converter Xilinx IP - RF-DAC")

On a la disposition des DACs sur la carte fille XM500.

| Tuile | DAC | Interface | Fréquence | Nom | Connecteur |
| :---: | :---: | :---: | :---: | :---: | :---: |
| 228 | 0 | Direct | 0 - 4GHz | DAC228_T0_CH0 | J26 / J27 |
| 228 | 1 | Direct | 0 - 4GHz | DAC228_T0_CH1 | J20 / J21 |
| 228 | 2 | Direct | 0 - 4GHz | DAC228_T0_CH2 | J22 / J23 |
| 228 | 3 | Direct | 0 - 4GHz | DAC228_T0_CH3 | J24 / J25 |
| 229 | 0 | Balun HF | 1 - 4 GHz | DAC229_T1_CH0 | J7 |
| 229 | 1 | Balun HF | 1 - 4 GHz | DAC229_T1_CH1 | J8 |
| 229 | 2 | Balun LF | 0 - 1 GHz | DAC229_T1_CH2 | J5 |
| 229 | 3 | Balun LF | 0 - 1 GHz | DAC229_T1_CH3 | J6 |

## Organisation des ADCs

Sur la ZCU111, les 8 ADCs sont disposés en 4 tuiles (224-225-226-227) de 2 ADCs chacune, organisées en paires (0-1).

![RF_ADC](./../images/RF_ADC.png?raw=true "Zynq UltraScale+ RFSoC RF Data Converter Xilinx IP - RF-ADC")

On a la disposition des ADCs sur la carte fille XM500.

| Tuile | ADC | Interface | Fréquence | Nom | Connecteur |
| :---: | :---: | :---: | :---: | :---: | :---: |
| 227 | 0 | Direct | 0 - 4GHz | ADC226_T2_CH0 | J32 / J33 |
| 227 | 1 | Direct | 0 - 4GHz | ADC226_T2_CH1 | J34 / J35 |
| 226 | 0 | Direct | 0 - 4GHz | ADC227_T3_CH0 | J36 / J37 |
| 226 | 1 | Direct | 0 - 4GHz | ADC227_T3_CH1 | J39 / J40 |
| 225 | 0 | Balun HF | 1 - 4 GHz | ADC225_T1_CH0 | J2 |
| 225 | 1 | Balun HF | 1 - 4 GHz | ADC225_T1_CH1 | J1 |
| 224 | 0 | Balun LF | 0 - 1 GHz | ADC224_T0_CH0 | J4 |
| 224 | 1 | Balun LF | 0 - 1 GHz | ADC224_T0_CH1 | J3 |

## Paramétrage des horloges

Les convertisseurs d'une même tuile partagent différents éléments.
En particulier, on peut paramétrer :

- la fréquence d'échantillonnage
- l'horloge de l'inteface AXI4-Stream
- la PLL qui permet de générer une horloge à une fréquence désirée si nécessaire
- l'horloge de sortie si la PLL est activée

La fréquence d'échantillonnage choisie détermine la fréquence de l'horloge de l'interface AXI4-Stream nécessaire.

Cette horloge peut être générée directement avec l'IP \textit{Zynq UltraScale+ RFSoC RF Data Converter} fournit par Xilinx, en exploitant les PLLs disponnibles.
La valeur par défaut des PLLs (409.600 MHz) est adaptée à cet exemple.
Il est cependant possible de la modifier, notamment si l'on souhaite utiliser une fréquence d'échantillonnage qui ne soit pas en base binaire (1.024 GSa/s), mais en base décimale (1 GSa/s).

Les convertisseurs d'une même tuile partagent les mêmes horloges, on peut donc être sûr qu'ils seront parfaitement synchronisés entre eux.
Il faut donc associer correctement les convertisseurs en privilégiant ceux qui sont dans la même tuile.

Il faut aussi s'assurer que les signaux numériques des deux canaux sont disponibles en même temps pour être convertis ensemble.
