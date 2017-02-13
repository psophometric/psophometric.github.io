## Découvrir LoRaWAN avec The Things Network et deux Raspberry

Difficile de passer à coté de l'_Internet des Objets_ (IoT). De nombreux articles sont régulièrement publiés sur le sujet. Les entreprises s'y mettent.
Des réseaux dédiés se construisent avec des technologies développées pour cela (Sigfox, LoRaWAN).
Et justement, la technologie LoRaWAN a retenu mon attention car il s'agit d'un protocole de communication accessible à tout le monde.

Comme je dispose déjà de deux Raspeberry, un modèle Pi 3 et un Pi Zero, j'ai décidé de les utiliser pour réaliser une communication LoRaWAN.
Il ne me reste qu'à commander deux modules radio Lora et à trouver des logiciels pour faire communiquer le tout.

Voici le récit de mes premiers pas dans le monde de l'_Internet des Objets._

Avant d'aller plus loin, je vous invite à consulter ces articles qui décrivent le fonctionnement de LoRaWAN et la terminologie associée :
- [A Closer Look at LoRaWAN and The Things Network, DesignSpark, en anglais](https://www.rs-online.com/designspark/a-closer-look-at-lorawan-and-the-things-network)
- [De la technologie LoRa au réseau LoRaWAN, Frugal Prototype, en français](http://www.frugalprototype.com/technologie-lora-reseau-lorawan/)

Le système LoRaWAN se compose : d'un objet connecté (_End-device_ ou _Node_ dans la terminologie LoRaWAN), d'un concentrateur (qui fait l'interface Lora / Internet, _Gateway_ dans la terminologie LoRaWAN), un réseau (celui de _The Things Network_, le fonctionnement y est décrit [ici](https://www.thethingsnetwork.org/wiki/Backend/Home)) qui reçoit les données et une application (ici _Node-Red_) qui exploite les données.

![Shield Lora Raspberry Pi 3 & Pi Zero](https://psophometric.github.io/decouvrir-ttn-lorawan/images/LW-shield-lora-raspberry.jpg)

### Gateway

La Gateway est réalisée avec un Raspberry Pi 3, un module radio RFM95W et un programme en C++.
Avec ce matériel, il ne s'agit pas d'une Gateway pleinement compatible avec LoRaWAN puisqu'elle ne propose que l'UpLink sur un seul canal, mais cela est suffisant pour réaliser des essais.

Les instructions pour réaliser la gateway se trouvent [ici](https://www.hackster.io/ChrisSamuelson/lora-raspberry-pi-single-channel-gateway-cheap-d57d36).

Après avoir installé [Single Channel LoRaWAN Gateway](https://github.com/tftelkamp/single_chan_pkt_fwd), il suffit d'éditer le fichier main.cpp, pour indiquer l'adresse du serveur TTN correspondant à la région déclarée lors de l'insription à The Things Network.
Dans mon cas : `#define SERVER1 "40.114.249.243"    //  router.eu.thethings.network`
Et de compiler avec `make`.
Ou d'utiliser le [Fork disponible sur mon Github](https://github.com/psophometric/single_chan_pkt_fwd), qui contient la modification.

A noter que si on utilise des Pin du GPIO différentes de celle indiqué dans le tutorial, il faut là aussi modifier dans main.cpp :
```
// Uputronics - Raspberry connections
int ssPin = 10;
int dio0  = 6;
int RST   = 0;
```

### Node

La Node est réalisée avec un Raspberry Pi Zero et un module radio RFM95W.

Le module radio RFM95W n'étant qu'un modem Lora, il faut ajouter une couche protocolaire LoRaWAN.
Pour cela, j'ai utilisé la librairie LMIC, adaptée au Raspberry : [arduino-lmic Branch:rpi](https://github.com/hallard/arduino-lmic/tree/rpi).

Comme j'utilise une "pseudo" Gateway "Single channel", il n'est pas possible d'utiliser l'activation OTAA (qui nécessite le DownLink et plusieurs canaux), mais uniquement l'activation ABP.
Or, dans la Branch:rpi de la librairie LMIC, l'activation ABP n'a pas été implémentée.
Qu'à cela ne tienne, en regardant dans la Branch:master, qui regroupe les programmes pour Arduino, on peut comparer les paramètres qui diffèrent entre les deux modes.
Dans le [Fork disponible sur mon Github](https://github.com/psophometric/arduino-lmic/tree/rpi/examples/raspi), on peut retrouver le résultat dans le dossier [TTN-ABP](https://github.com/psophometric/arduino-lmic/tree/rpi/examples/raspi/ttn-abp).
Il suffit de mettre les valeurs de DevAddr, NwkSKey et AppSKey générées depuis la console TTN pour que le programme soit utilisable (après avoir compilé avec `make`).

Un Hello, world! c'est bien, mais pas très intéractif...

![The Things Network - Hello world](https://psophometric.github.io/decouvrir-ttn-lorawan/images/LW-Helloworld_TTN.jpeg)

Je n'ai pas de capteurs (température, humidité, accéléromètre, ...) mais comme beaucoup de monde, j'ai un smartphone !
Et ce smartphone est équipé d'un GPS : ce qui peut être interressant ! Comment récupérer les données GPS du smartphone depuis le Raspberry ?
Pour cela, il suffit d'installer sous Android : "Share GPS" et de créer une connection TCP/IP pour envoyer les informations du GPS par la connection WiFi.

Bien sûr, il faut ajouter un bundle WiFi au Raspberry Pi Zero et configurer le réseau WiFi du smarthone en mode _Point d'accès mobile._
Sur le téléphone, j'installe aussi l'application JuiceSSH afin de pouvoir me connecter au Raspberry en SSH.

Ensuite, il faut modifier le programme TTN-ABP pour qu'il transmette les coordonnées GPS reçues.
N'étant pas très à l'aise avec le C++, j'ai écris un programme Python qui reçoit et renvoit par la ligne de commande (stdout) longitude et latitude du GPS.
Dans le programme TTN-ABP j'ai juste ajouter un "cin" pour récupérer les coordonnées par la ligne de commande (stdin).

Les coordonnées sont reçues sous la forme : ll.llll,L.LLLL (correspondant à latitude, Longitude).
Il me reste deux fonctions à réaliser afin de pouvoir utiliser le programme :
- dans le programme en C++ TTN-ABP : récupérer latitude et longitude dans des variables
- et définir un format pour les transmettre (ascii? flottant? ...)

En cherchant un peu sur Internet, j'ai pu réaliser ces modifications avec de simples copier-coller :
- Ajout d'une fonction pour séparer les informations de latitude et longitude (split ','). [Fonction à retrouver ici](https://www.safaribooksonline.com/library/view/c-cookbook/0596007612/ch04s07.html)
- Utilisation de nombres flottants [d'après une discussion sur le forum TTN](https://www.thethingsnetwork.org/forum/t/best-practices-when-sending-gps-location-data/1242/13)

Ces copier-coller sont l'occasion de faire remarquer l'apport des contributions libres. En effet, sans connaissances approfondies en programmation, j'ai pu faire aboutir mon projet en réutilisant les fonctions écrites par d'autres.

Bref, pour vous éviter tous ces copier-coller, j'apporte contribution en mettant à disposition le résultat sur le [Fork disponible sur mon Github](https://github.com/psophometric/arduino-lmic/tree/rpi/examples/raspi), on peut retrouver le résultat dans le dossier [TTN-ABP-GPS](https://github.com/psophometric/arduino-lmic/tree/rpi/examples/raspi/ttn-abp-gps).


### The Things network
Sur le site _The Things Network,_ il existe une [documentation](https://www.thethingsnetwork.org/docs/) qui explique très bien comment créer une application.
Voir également [ce lien](https://github.com/TheThingsNetwork/workshops).
Pour traiter le format des données reçues, après avoir créé l'application, dans l'onglet "Payload Functions", ajouter les lignes indiquées [sur le forum TTN](https://www.thethingsnetwork.org/forum/t/best-practices-when-sending-gps-location-data/1242/13)

![The Things Network - Payload Functions](https://psophometric.github.io/decouvrir-ttn-lorawan/images/LW-TTN-PayloadF.png)

### Application : Node-Red
La dernière étape est l'installation de Node-Red.
On peut l'installer sur un PC ou directement sur le Raspberry qui héberge la Gateway. J'ai choisi cette deuxième possibilité.
Voir [cet article en Français](http://www.projetsdiy.fr/node-red-decouverte-sur-raspberry-pi-3-ou-2/).

L'application que je souhaite créer doit récupérer les données depuis le serveur _The Things Network_ et afficher les points GPS sur une carte _OpenStreetMap._
Pour cela, il faut installer les compléments à Node-Red suivants :
- [node-red-contrib-ttn](http://flows.nodered.org/node/node-red-contrib-ttn)
- [node-red-contrib-web-worldmap](http://flows.nodered.org/node/node-red-contrib-web-worldmap)

Ensuite, je réalise un _flow_ minimaliste :

![Node-Red Flow](https://psophometric.github.io/decouvrir-ttn-lorawan/images/LW-NODERED_Flow.png)

Et configure chaque élément :

The Things Network : il suffit d'entrer le nom de l'application ainsi que l'Access Key que l'on récupère sur la console The Things Network.
![Node-Red The Things Network](https://psophometric.github.io/decouvrir-ttn-lorawan/images/LW-NODERED_TTN.png)

La fonction permet de créer trois champs dans le Payload du message : Name, Lat et Lon qui seront utilisés par Worldmap
![Node-Red Function](https://psophometric.github.io/decouvrir-ttn-lorawan/images/LW-NODERED_FunctionPoint.png)

Worldmap : il suffit d'indiquer la latitude et longitude sur lesquelles sera centrée la carte à l'ouverture de la page.
![Node-Red Web Worldmap](https://psophometric.github.io/decouvrir-ttn-lorawan/images/LW-NODERED_mymap.png)


Pour commencer l'envoi des coordonnées GPS, il n'y a plus qu'à lancer depuis le Raspberry Pi Zero (Node) :
```
cd arduino-lmic/examples/raspi/ttn-abp-gps/
python nmea.py | sudo ./ttn-abp-gps
```

Et à se connecter à Node-Red depuis un navigateur :
![Node-Red Web Worldmap The Things Network](https://psophometric.github.io/decouvrir-ttn-lorawan/images/LW-TTN-ABP-GPS.jpeg)

En ajoutant quelques lignes de code, on peut personnaliser l'affichage avec des lignes, des couleurs en fonction du RSSI (niveau de puissance reçu du signal par la Gateway), ...
![Node-Red The Things Network Line iconColor](https://psophometric.github.io/decouvrir-ttn-lorawan/LW-NODERED_Line_iconColor.png)

### Que conclure ?

Que ce projet n'a la prétention que d'être une première initiation à la découverte de ce qu'il est possible de faire autour de LoRaWAN. Bien sûr pour réaliser un traqueur GPS fonctionnel, il faudrait privilégier la création d'une _Node_ à partir d'un Arduino et avec un récepteur GPS autonome. Cela serait l'occasion de mettre en oeuvre un montage sur batterie et de chercher à optimiser l'autonomie.

### Fonctionnement en vidéo à voir sur Twitter
<blockquote class="twitter-tweet" data-lang="fr"><p lang="fr" dir="ltr">Cette fois, je fais une balade à pied pour voir voir la couverture radio <a href="https://twitter.com/hashtag/LoRaWan?src=hash">#LoRaWan</a> grâce aux données GPS - <a href="https://twitter.com/hashtag/NodeRED?src=hash">#NodeRED</a> <a href="https://twitter.com/TTN_Paris">@TTN_Paris</a> <a href="https://t.co/1ak3Ly03Bu">pic.twitter.com/1ak3Ly03Bu</a></p>&mdash; psopho (@psophometric) <a href="https://twitter.com/psophometric/status/830459988879503360">11 février 2017</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>
