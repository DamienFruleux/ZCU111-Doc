![AWG](./../images/AWG.png?raw=true "Architecture de l'AWG")
![DGTZ](./../images/DGTZ.png?raw=true "Architecture du DGTZ")

# Le protocole AXI4

L' AXI4 ([Advanced eXtensible Interface 4](https://en.wikipedia.org/wiki/Advanced_eXtensible_Interface)) est une interface de communication parallèle, principalement conçue pour la communication sur puce.

Il existe trois types d'interfaces AXI4 ([UG761](https://docs.xilinx.com/v/u/en-US/ug761_axi_reference_guide)) :

- **AXI4 Memory Map** : pour les interfaces à mémoire mappée.
**Elle permet des rafales de 256 cycles de transfert de données avec une seule phase d'adressage**.
Utile pour les exigences de haute performance.

- **AXI4-Lite** : interface légère, **à transaction unique en mémoire**.
Elle présente une empreinte logique réduite et constitue une interface simple à utiliser, tant au niveau de la conception que de l'utilisation.
Elle est utile pour les communications simples et à faible débit en mémoire (par exemple, vers et depuis les registres de contrôle et d'état).

- **AXI4-Stream** : **supprime la nécessité d'une phase d'adressage et permet une taille illimitée des burst de données**.
Les interfaces et les transferts AXI4-Stream n'ont pas de phase d'adressage et ne sont donc pas considérés comme étant mappées en mémoire.
Utile pour les flux de données à grande vitesse (j'aime parler de "transfert temps réel" pour illustrer la différence avec des transferts par paquets comme l'AXI4 Memory Map).

En pratique, pour les interfaces AXI4 Memory Map et AXI4-Lite, il est nécessaire de lire ou d'écrire dans les registres de configuration aux bonnes adresses.
Pour exploiter l'interface AXI4-Stream, il faut utiliser une DMA.

# Définition de la DMA

Une DMA ([Direct Memory Access](https://en.wikipedia.org/wiki/Direct_memory_access)) est un dispositif qui permet un accès direct entre la RAM et un périphérique (comme un HDD, un SSD...), sans l'intervention du processeur, sauf pour démarrer et arrêter le transfert.
Cela accélère donc considérablement les transferts de données.

Dans la nomenclature de Xilinx, on retrouve des DMA avec la même définition, mais qui permettent plus précisément de transférer des données entre une interface AXI4 Memory Map, qui est connectée à la RAM SODIMM, ou **PS RAM**, et une interface AXI4-Stream, qui est connectée à un **périphérique externe**, tel qu'un DAC ou un ADC.

A noter qu'il est également possible d'utiliser une RAM externe, notamment les 4 Go de DDR4 SDRAM, ou **PL RAM**.
Cette mémoire vive utilise également une interface AXI4 Memory Map.

Pour la suite de ces explications, nous ne considérerons que la DMA au sens de Xilinx.

En particulier, la DMA dispose de 2 canaux distincts :

- Le canal de lecture : **lecture des données depuis la RAM** et écriture dans le périphérique.

- Le canal d'écriture : lecture des données depuis le périphérique et **ecriture dans la RAM**.

On utilise l'IP (Intellectual Property) [AXI DMA Controller](https://www.xilinx.com/products/intellectual-property/axi_dma.html) fournit par Xilinx afin de transférer des données entre la PS (Processing System : CPU ARM) et la PL (Programable Logical : FPGA).

![AWG_DMA](./../images/AWG_DMA.png?raw=true "Disposition de la DMA dans l'architecture de l'AWG")
![DGTZ_DMA](./../images/DGTZ_DMA.png?raw=true "Disposition de la DMA dans l'architecture du DGTZ")

# Caractéristiques techniques de la DMA

On retrouve les principales caractéristiques techniques de l'IP [AXI DMA Controller](https://www.xilinx.com/products/intellectual-property/axi_dma.html) fournit par Xilinx.

![DMA](./../images/DMA.png?raw=true "IP Xilinx AXI DMA Controller")

## Taille maximale d'un transfert

La largeur maximale du registre de longueur de la mémoire tampon est de **2<sup>26</sup>-1 octets**.
Cela signifie qu'un transfert DMA ne peut pas contenir plus de **64 Mo - 1** de données (il s'agit d'une limite matérielle de l'IP de Xilinx).

> Plus précisément : 2<sup>26</sup> bytes = 67 108 864 octets = 64 Mo

Il est important de le savoir car en principe, on ne peut pas faire un transfert DMA de plus de 64 Mo - 1, en particulier si on utilise la librairie Pynq.
Une méthode pour contourner cette limite est proposée par la suite.

Attention, vivado n'est pas très clair dans la configuration, car elle nécessite un nombre de bits ([PG021](https://docs.xilinx.com/r/en-US/pg021_axi_dma)) : "*The number of bytes is equal to **2<sup>Length Width</sup>** . So a Length Width of 26 gives a byte count of **67,108,863 bytes**.* "

## Débit maximale d'un transfert

Le débit d'un transfert dépend des différents paramètres de l'interface considérée : en particulier la PS RAM et la PL RAM ont des caractéristiques différentes.

### Côté AXI4 Memory Map, utilisation de la PS RAM

Du côté AXI4 Memory Map, le débit est limitée par la PS : la largeur des bus de données AXI4 HP0/1/2/3 est limitée à **128 bits**.
Sachant que la fréquence d'horloge maximale de la PS est de **333 MHz**, le taux de transfert maximal du côté AXI4 Memory Map est de **5,20 Go/s**.

> Plus précisément : 128 bits * 333 MHz = 42 624 * 10<sup>6</sup> bps = 41.625 Gbps = 5.20 Go/s

Avec une horloge plus classique de **300 MHz**, on atteint un taux de transfert de **4.6875 Go/s**.

> Plus précisément : 128 bits * 300 MHz = 38 400 * 10<sup>6</sup> bps = 37.5 Gbps = 4.6875 Go/s

De plus, HP1 et HP2 partagent la même interface AXI4 Memory Map 128 bits ([UG1085, figure 1-1, page 29](https://www.xilinx.com/support/documentation/user_guides/ug1085-zynq-ultrascale-trm.pdf)) de sorte que le débit ne sera pas maximale s'ils sont utilisés en même temps.

#### En utilisant des "Samples per seconds"

Par la suite, nous utiliserons une DMA pour envoyer ou recevoir des données avec des DACs et des ADCs.
Généralement, on parle en GSa/s (Samples per seconds, ou GSPS dans la nomenclature Xilinx) plutôt qu'en Go/s.

Comme nous le verrons, les DACs et ADCs utilisent des données codées sur 16 bits, donc **1 échantillon = 16 bits = 2 octets**.

Le taux de transfert maximal de ce côté est donc également de **2,6 GSa/s** avec une horloge de 333 MHz ou **2,34375 GSa/s** avec une horloge plus classique de 300 MHz.

Il est préférable d'utiliser une horloge de 300 MHz, car en plus d'être plus classique, c'est à cette fréquence que fonctionne la PL RAM : cela permet de réutiliser facilement le même design entre la PS RAM et la PL RAM et donc augmenter la compatibilité.
Dans le cas ou l'on souhaite les performances maximales, il reste 33MHz de marge.

### Côté AXI4 Memory Map, utilisation de la PL RAM

Comme nous l'avons vu plus haut, il est également possible d'utiliser les 4 Go de DDR4 SDRAM (PL RAM).
Dans ce cas, la largeur du bus de données est de **512 bits**.
De plus, la mémoire DDR4 SDRAM nécessite une fréquence d'horloge de **300 MHz**.
Dans ce cas, le taux de transfert maximal du côté AXI4 Memory Map est de **18,75 Go/s**.
							
> Plus précisément : 512 bits * 300 MHz = 153 600 * 10<sup>6</sup> bps = 150 Gbps = 18.75 Go/s

#### En utilisant des "Samples per seconds"

Le taux de transfert maximal du côté AXI4 Memory Map est également de **9,375 GSa/s**.

### Côté AXI4-Stream

La largeur des données de l'interface AXI4-Stream de la DMA peut être de 8, 16, 32, 64, 128, 256, 512 et 1024 bits ([PG021](https://docs.xilinx.com/r/en-US/pg021_axi_dma))
Il s'agit du nombre de bits qui seront envoyés ou reçus par la DMA en 1 coup d'horloge, mais cela ne correspond pas à la quantité totale de données transférées par la DMA, qui est de 64 Mo-1, comme nous l'avons vu précédemment.

De l'autre côté de l'interface, le débit est définie par l'IP utilisée : généralement, il n'est pas possible de choisir directement ce paramètre puisqu'il dépends du fonctionnement de cette IP.
Dans notre cas, les DACs fonctionnent avec une largeur de données de 256 bits, tandis que les ADCs fonctionnent avec une largeur de données de 128 bits.

## Quantité maximale de mémoire

La ZCU111 dispose de 2 RAMs différentes : la PS RAM et la PL RAM.

Il est possible de changer la barrette de RAM SODIMM (([DS889](https://docs.xilinx.com/v/u/en-US/ds889-zynq-usp-rfsoc-overview))), afin de passer de 4 Go (par défaut) à **32 Go de mémoire dans la PS**.
En théorie cela correspond à respectivement 2 GSa et 16 GSa, mais il faut garder en tête que le noyau GNU/Linux utilise aussi cette mémoire !
La quantitée de mémoire réellement utilisable n'est ainsi pas parfaitement définie.
Enfin, on ne peut pas gérer cette mémoire comme on le souhaite : il faut faire des allocations de mémoire.

Malheureusement, il n'est pas possible d'augmenter la DDR4 SDRAM, car ces puces mémoire sont soudées.
On dispose de **4 Go de mémoire dans la PL**.
De plus, celle-ci n'est pas du tout utilisée par le noyau, on à ainsi 2 GSa disponnible que l'on peut utiliser et gérer à notre convenance.

On a donc un compromis à faire entre le débit et la quantité de mémoire : **~ 32 Go @ 5.20 Go/s ou 4 Go @ 18.75 Go/s !**
