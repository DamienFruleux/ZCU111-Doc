# Le protocole AXI4-Stream

Le protocole [AXI4-Stream](https://wiki.electroniciens.cnrs.fr/index.php/FPGA_CPLD_:_Guides_:_AXI4-Stream) est un protocole avec ``handshake'' qui utilise entre 2 et 9 signaux pour communiquer, bien que 3 à 4 signaux soient généralement suffisants et nécessaires (figure \ref{figure_axi4-stream}).
Il s'agit d'un protocole ``maître-esclave'' : le maître envoie les données et l'esclave les reçoit.

![AXI4-Stream](./images/AXI4-Stream.png?raw=true "AXI4-Stream Schema")

Les deux signaux suivants sont obligatoires :

- *TDATA* : vecteur de données
- *TVALID* : indique qu'une donnée est présente et valide pour l'esclave

Dans la plupart des cas, un signal supplémentaire est utilise :

- *TREADY* : présenté par l'esclave pour indiquer qu'il est prêt à recevoir les données

Il est possible de se passer du signal TREADY et dans ce cas, la transmission est considérée comme ayant lieu sans confirmation de la part de l'esclave.

Le signal suivant est également couramment utilisé :

- *TLAST* : indique la fin d'une trame, utile lorsque les trames sont de taille variable

Une trame est définie comme l'ensemble des données envoyées par le maître vers l'esclave, en une seule itération.
Il est donc possible d'envoyer plusieurs trames lors d'un même transfert.
Il est alors nécessaire d'ajuster le signal TLAST.

Le canal de lecture (**lecture des données depuis la RAM** et écriture dans le périphérique) de la DMA utilise les 3 premiers signaux pour envoyer le signal au DAC, alors que le canal d'écriture (lecture des données depuis périphérique et **ecriture dans la RAM**) impose en plus le signal TLAST pour recevoir les signaux de l'ADC.

Le nombre de trames souhaité est variable : en particulier si l'on veut avoir un signal continu !

# Convertisseurs RF

On retrouve différent convertisseurs RF (Radio-Frequence) que l'on souhaite exploiter sur la ZCU111.
Un [DAC](https://en.wikipedia.org/wiki/Digital-to-analog_converter) (Digital to Analog Converter) est un dispositif qui convertit un signal numérique en un signal analogique tandis qu'un [ADC](https://en.wikipedia.org/wiki/Analog-to-digital_converter) (Analog to Digital Converter) est un dispositif qui convertit un signal analogique en un signal numérique.

On utilise l'IP [Zynq UltraScale+ RFSoC RF Data Converter](https://www.xilinx.com/products/intellectual-property/rf-data-converter.html) fournit par Xilinx afin de configurer les DACs et les ADCs et de les interfacer à la DMA au moyen du protocole AXI4-Stream.
