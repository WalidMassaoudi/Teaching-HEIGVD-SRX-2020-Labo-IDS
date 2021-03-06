# Teaching-HEIGVD-SRX-2020-Laboratoire-IDS

**Ce travail de laboratoire est à faire en équipes de 2 personnes** (oui... en remote...). Je vous laisse vous débrouiller ;-)

**ATTENTION : Commencez par créer un Fork de ce repo et travaillez sur votre fork.**

Clonez le repo sur votre machine. Vous pouvez répondre aux questions en modifiant directement votre clone du README.md ou avec un fichier pdf que vous pourrez uploader sur votre fork.

**Le rendu consiste simplement à répondre à toutes les questions clairement identifiées dans le text avec la mention "Question" et à les accompagner avec des captures. Le rendu doit se faire par une "pull request". Envoyer également le hash du dernier commit et votre username GitHub par email au professeur et à l'assistant**

## Table de matières

[Introduction](#introduction)

[Echéance](#echéance)

[Configuration du réseau](#configuration-du-réseau-sur-virtualbox)

[Installation de Snort](#installation-de-snort-sur-linux)

[Essayer Snort](#essayer-snort)

[Utilisation comme IDS](#utilisation-comme-un-ids)

[Ecriture de règles](#ecriture-de-règles)

[Travail à effectuer](#exercises)


## Echéance 

Ce travail devra être rendu le dimanche après la fin de la 2ème séance de laboratoire, soit au plus tard, **le 6 avril 2020, à 23h59.**


## Introduction

Dans ce travail de laboratoire, vous allez explorer un système de detection contre les intrusions (IDS) dont l'utilisation es très répandue grace au fait qu'il est gratuit et open source. Il s'appelle [Snort](https://www.snort.org). Il existe des versions de Snort pour Linux et pour Windows.

### Les systèmes de detection d'intrusion

Un IDS peut "écouter" tout le traffic de la partie du réseau où il est installé. Sur la base d'une liste de règles, il déclenche des actions sur des paquets qui correspondent à la description de la règle.

Un exemple de règle pourrait être, en language commun : "donner une alerte pour tous les paquets envoyés par le port http à un serveur web dans le réseau, qui contiennent le string 'cmd.exe'". En on peut trouver des règles très similaires dans les règles par défaut de Snort. Elles permettent de détecter, par exemple, si un attaquant essaie d'executer un shell de commandes sur un serveur Web tournant sur Windows. On verra plus tard à quoi ressemblent ces règles.

Snort est un IDS très puissant. Il est gratuit pour l'utilisation personnelle et en entreprise, où il est très utilisé aussi pour la simple raison qu'il est l'un des plus efficaces systèmes IDS.

Snort peut être exécuté comme un logiciel indépendant sur une machine ou comme un service qui tourne après chaque démarrage. Si vous voulez qu'il protège votre réseau, fonctionnant comme un IPS, il faudra l'installer "in-line" avec votre connexion Internet. 

Par exemple, pour une petite entreprise avec un accès Internet avec un modem simple et un switch interconnectant une dizaine d'ordinateurs de bureau, il faudra utiliser une nouvelle machine executant Snort et placée entre le modem et le switch. 


## Matériel

Vous avez besoin de votre ordinateur avec VirtualBox et une VM Kali Linux. Vous trouverez un fichier OVA pour la dernière version de Kali sur `//eistore1/cours/iict/Laboratoires/SRX/Kali` si vous en avez besoin.


## Configuration du réseau sur VirtualBox

Votre VM fonctionnera comme IDS pour "protéger" votre machine hôte (par exemple, si vous faites tourner VirtualBox sur une machine Windows, Snort sera utilisé pour capturer le trafic de Windows vers l'Internet).

Pour cela, il faudra configurer une réseau de la VM en mode "bridge" et activer l'option "Promiscuous Mode" dans les paramètres avancés de l'interface. Le mode bridge dans l'école ne vous permet pas d'accéder à l'Internet depuis votre VM. Vous pouvez donc rajouter une deuxième interface réseau à votre Kali configurée comme NAT. La connexion Internet est indispensable pour installer Snort mais pas vraiment nécessaire pour les manipulations du travail pratique.

Pour les captures avec Snort, assurez-vous de toujours indiquer la bonne interface dans la ligne de commandes, donc, l'interface configurée en mode promiscuous.

![Topologie du réseau virtualisé](images/Snort_Kali.png)


## Installation de Snort sur Linux

On va installer Snort sur Kali Linux. Si vous avez déjà une VM Kali, vous pouvez l'utiliser. Sinon, vous avez la possibilité de copier celle sur `eistore`.

La manière la plus simple c'est de d'installer Snort en ligne de commandes. Il suffit d'utiliser la commande suivante :

```
sudo apt update && apt install snort
```

Ceci télécharge et installe la version la plus récente de Snort.

Vers la fin de l'installation, on vous demande de fournir l'adresse de votre réseau HOME. Il s'agit du réseau que vous voulez protéger. Cela sert à configurer certaines variables pour Snort. Pour les manipulations de ce laboratoire, vous pouvez donner n'importe quelle adresse comme réponse.


## Essayer Snort

Une fois installé, vous pouvez lancer Snort comme un simple "sniffer". Pourtant, ceci capture tous les paquets, ce qui peut produire des fichiers de capture énormes si vous demandez de les journaliser. Il est beaucoup plus efficace d'utiliser des règles pour définir quel type de trafic est intéressant et laisser Snort ignorer le reste.

Snort se comporte de différentes manières en fonction des options que vous passez en ligne de commande au démarrage. Vous pouvez voir la grande liste d'options avec la commande suivante :

```
snort --help
```

On va commencer par observer tout simplement les entêtes des paquets IP utilisant la commande :

```
snort -v -i eth0
```

**ATTENTION : assurez-vous de bien choisir l'interface qui se trouve en mode bridge/promiscuous. Elle n'est peut-être pas eth0 chez-vous!**

Snort s'execute donc et montre sur l'écran tous les entêtes des paquets IP qui traversent l'interface eth0. Cette interface est connectée à l'interface réseau de votre machine hôte à travers le bridge de VirtualBox.

Pour arrêter Snort, il suffit d'utiliser `CTRL-C` (**attention** : il peut arriver de temps à autres que snort ne réponde pas correctement au signal d'arrêt. Dans ce cas-là, il faudra utiliser `kill` pour arrêter le process).

## Utilisation comme un IDS

Pour enregistrer seulement les alertes et pas tout le trafic, on execute Snort en mode IDS. Il faudra donc spécifier un fichier contenant des règles. 

Il faut noter que `/etc/snort/snort.config` contient déjà des références aux fichiers de règles disponibles avec l'installation par défaut. Si on veut tester Snort avec des règles simples, on peut créer un fichier de config personnalisé (par exemple `mysnort.conf`) et importer un seul fichier de règles utilisant la directive "include".

Les fichiers de règles sont normalement stockes dans le repertoire `/etc/snort/rules/`, mais en fait un fichier de config et les fichiers de règles peuvent se trouver dans n'importe quel repertoire. 

Par exemple, créez un fichier de config `mysnort.conf` dans le repertoire `/etc/snort` avec le contenu suivant :

```
include /etc/snort/rules/icmp2.rules
```

Ensuite, créez le fichier de règles `icmp2.rules` dans le repertoire `/etc/snort/rules/` et rajoutez dans ce fichier le contenu suivant :

`alert icmp any any -> any any (msg:"ICMP Packet"; sid:4000001; rev:3;)`

On peut maintenant executer la commande :

```
snort -c /etc/snort/mysnort.conf
```

Vous pouvez maintenant faire quelques pings depuis votre hôte et regarder les résultas dans le fichier d'alertes contenu dans le repertoire `/var/log/snort/`. 


## Ecriture de règles

Snort permet l'écriture de règles qui décrivent des tentatives de exploitation de vulnérabilités bien connues. Les règles Snort prennent en charge à la fois, l'analyse de protocoles et la recherche et identification de contenu.

Il y a deux principes de base à respecter :

* Une règle doit être entièrement contenue dans une seule ligne
* Les règles sont divisées en deux sections logiques : (1) l'entête et (2) les options.

L'entête de la règle contient l'action de la règle, le protocole, les adresses source et destination, et les ports source et destination.

L'option contient des messages d'alerte et de l'information concernant les parties du paquet dont le contenu doit être analysé. Par exemple:

```
alert tcp any any -> 192.168.1.0/24 111 (content:"|00 01 86 a5|"; msg: "mountd access";)
```

Cette règle décrit une alerte générée quand Snort trouve un paquet avec tous les attributs suivants :

* C'est un paquet TCP
* Emis depuis n'importe quelle adresse et depuis n'importe quel port
* A destination du réseau identifié par l'adresse 192.168.1.0/24 sur le port 111

Le text jusqu'au premier parenthèse est l'entête de la règle. 

```
alert tcp any any -> 192.168.1.0/24 111
```

Les parties entre parenthèses sont les options de la règle:

```
(content:"|00 01 86 a5|"; msg: "mountd access";)
```

Les options peuvent apparaître une ou plusieurs fois. Par exemple :

```
alert tcp any any -> any 21 (content:"site exec"; content:"%"; msg:"site
exec buffer overflow attempt";)
```

La clé "content" apparait deux fois parce que les deux strings qui doivent être détectés n'apparaissent pas concaténés dans le paquet mais à des endroits différents. Pour que la règle soit déclenchée, il faut que le paquet contienne **les deux strings** "site exec" et "%". 

Les éléments dans les options d'une règle sont traitées comme un AND logique. La liste complète de règles sont traitées comme une succession de OR.

## Informations de base pour le règles

### Actions :

```
alert tcp any any -> any any (msg:"My Name!"; content:"Skon"; sid:1000001; rev:1;)
```

L'entête contient l'information qui décrit le "qui", le "où" et le "quoi" du paquet. Ça décrit aussi ce qui doit arriver quand un paquet correspond à tous les contenus dans la règle.

Le premier champ dans le règle c'est l'action. L'action dit à Snort ce qui doit être fait quand il trouve un paquet qui correspond à la règle. Il y a six actions :

* alert - générer une alerte et écrire le paquet dans le journal
* log - écrire le paquet dans le journal
* pass - ignorer le paquet
* drop - bloquer le paquet et l'ajouter au journal
* reject - bloquer le paquet, l'ajouter au journal et envoyer un `TCP reset` si le protocole est TCP ou un `ICMP port unreachable` si le protocole est UDP
* sdrop - bloquer le paquet sans écriture dans le journal

### Protocoles :

Le champ suivant c'est le protocole. Il y a trois protocoles IP qui peuvent être analysez par Snort : TCP, UDP et ICMP.


### Adresses IP :

La section suivante traite les adresses IP et les numéros de port. Le mot `any` peut être utilisé pour définir "n'import quelle adresse". On peut utiliser l'adresse d'une seule machine ou un block avec la notation CIDR. 

Un opérateur de négation peut être appliqué aux adresses IP. Cet opérateur indique à Snort d'identifier toutes les adresses IP sauf celle indiquée. L'opérateur de négation est le `!`.

Par exemple, la règle du premier exemple peut être modifiée pour alerter pour le trafic dont l'origine est à l'extérieur du réseau :

```
alert tcp !192.168.1.0/24 any -> 192.168.1.0/24 111
(content: "|00 01 86 a5|"; msg: "external mountd access";)
```

### Numéros de Port :

Les ports peuvent être spécifiés de différentes manières, y-compris `any`, une définition numérique unique, une plage de ports ou une négation.

Les plages de ports utilisent l'opérateur `:`, qui peut être utilisé de différentes manières aussi :

```
log udp any any -> 192.168.1.0/24 1:1024
```

Journaliser le traffic UDP venant d'un port compris entre 1 et 1024.

--

```
log tcp any any -> 192.168.1.0/24 :6000
```

Journaliser le traffic TCP venant d'un port plus bas ou égal à 6000.

--

```
log tcp any :1024 -> 192.168.1.0/24 500:
```

Journaliser le traffic TCP venant d'un port privilégié (bien connu) plus grand ou égal à 500 mais jusqu'au port 1024.


### Opérateur de direction

L'opérateur de direction `->`indique l'orientation ou la "direction" du trafique. 

Il y a aussi un opérateur bidirectionnel, indiqué avec le symbole `<>`, utile pour analyser les deux côtés de la conversation. Par exemple un échange telnet :

```
log 192.168.1.0/24 any <> 192.168.1.0/24 23
```

## Alertes et logs Snort

Si Snort détecte un paquet qui correspond à une règle, il envoie un message d'alerte ou il journalise le message. Les alertes peuvent être envoyées au syslog, journalisées dans un fichier text d'alertes ou affichées directement à l'écran.

Le système envoie **les alertes vers le syslog** et il peut en option envoyer **les paquets "offensifs" vers une structure de repertoires**.

Les alertes sont journalisées via syslog dans le fichier `/var/log/snort/alerts`. Toute alerte se trouvant dans ce fichier aura son paquet correspondant dans le même repertoire, mais sous le fichier `snort.log.xxxxxxxxxx` où `xxxxxxxxxx` est l'heure Unix du commencement du journal.

Avec la règle suivante :

```
alert tcp any any -> 192.168.1.0/24 111
(content:"|00 01 86 a5|"; msg: "mountd access";)
```

un message d'alerte est envoyé à syslog avec l'information "mountd access". Ce message est enregistré dans `/var/log/snort/alerts` et le vrai paquet responsable de l'alerte se trouvera dans un fichier dont le nom sera `/var/log/snort/snort.log.xxxxxxxxxx`.

Les fichiers log sont des fichiers binaires enregistrés en format pcap. Vous pouvez les ouvrir avec Wireshark ou les diriger directement sur la console avec la commande suivante :

```
tcpdump -r /var/log/snort/snort.log.xxxxxxxxxx
```

Vous pouvez aussi utiliser des captures Wireshark ou des fichiers snort.log.xxxxxxxxx comme source d'analyse por Snort.

## Exercises

**Réaliser des captures d'écran des exercices suivants et les ajouter à vos réponses.**

### Essayer de répondre à ces questions en quelques mots :

**Question 1: Qu'est ce que signifie les "preprocesseurs" dans le contexte de Snort ?**

---

**Reponse :**  

Snort a plusieurs composants autres que le moteur de règles. Par exemple, certains paquets et applications doivent être décodés
en texte brut pour que les règles Snort se déclenchent. Le composant qui gère les paquets avant qu'ils n'atteignent le moteur
de règles est appelé le préprocesseur.

--- Source :https://www.oreilly.com/library/view/snort-cookbook/0596007914/ch04.html


---

**Question 2: Pourquoi êtes vous confronté au WARNING suivant `"No preprocessors configured for policy 0"` lorsque vous exécutez la commande `snort` avec un fichier de règles ou de configuration "home-made" ?**

---

**Reponse :**  

---  Le "Warning No preprocessors configured for policy 0" est dü à la configuration des préprocesseurs ,puisque notre règle est "home-     made ",la configuration des préprocessuers de base ne couvre pas la règle ajouté . 

--

### Trouver votre nom :

Considérer la règle simple suivante:

alert tcp any any -> any any (msg:"Mon nom!"; content:"Rubinstein"; sid:4000015; rev:1;)

**Question 3: Qu'est-ce qu'elle fait la règle et comment ça fonctionne ?**

---

**Reponse :**  

---

Une règle contient un ensemble des champs permettent de détecter ou bien manipuler une paquet .Au début Snort applique cette règle  sur les paquets capturées pour obtenir une permiere plage de paquet correspondant au type du protocole , adresse ip destination et adresse ip source (mentionés dans l'entète de la règle ) . La régle au dessus va créer une alerte sous le nom  "mon nom " pour tous les paquets TCP conntenant le texte "Rubinstein". 


Utiliser un éditeur et créer un fichier `myrules.rules` sur votre répertoire home. Rajouter une règle comme celle montrée avant mais avec votre nom ou un mot clé de votre préférence. Lancer snort avec la commande suivante :

```
sudo snort -c myrules.rules -i eth0
```

**Question 4: Que voyez-vous quand le logiciel est lancé ? Qu'est-ce que tous les messages affichés veulent dire ?**

---

**Reponse :**  
![Capture écran des messages de lancement du snort ](images/myrules.png)
![contenu de notre règle ](images/myrulesTCP.png)

Explications : Les messages affichés donnent une idée sur les règles chargées par la commande ainsi quelques informations détaillées .nous trouvons aussi les port de sortie et d'entrée ,le nombre de pattern qui matche notre  règle .aussi à la fin d'exécution de notre commande snort nous permet d'avoir une vue complète sur le nombre des paquets capturées ainsi les alertes générées .

---

Aller à un site web contenant dans son text votre nom ou votre mot clé que vous avez choisi (il faudra chercher un peu pour trouver un site en http...).

**Question 5: Que voyez-vous sur votre terminal quand vous visitez le site ?**

---

**Reponse :**

---
Puisque on a pas du préprocessuer configuré pour cette règle on observe le warning "No preprocessors configured for policy 0" ,par contre notre règle à trouvé le mot recherché dans notre site web .
![contenu de notre règle ](images/visitWeb.png)
Arrêter Snort avec `CTRL-C`.

**Question 6: Que voyez-vous quand vous arrêtez snort ? Décrivez en détail toutes les informations qu'il vous fournit.**

---

**Reponse :**  

--- 
Lorsque on arréte Snort quelques informations s'affichent à l'ècran. Il sont des récapulatifs sur les résultat du traitement des paquets capturés . Ces informations sont expliquées comme ce dessous :
* " Timing Statistics ": fournit des statistiques de synchronisation de base. Il comprend le nombre total de secondes et de paquets ainsi que les taux de traitement des paquets
* " Packet I/O Totals " : Cette section montre l'acquisition de paquets de base. Si vous lisez des pcaps, les totaux sont pour tous les pcaps combinés.
* " Memory usage summary " : Contient des informations concernant l'utilisation de l'espace memoire .
* " Protocol Statistics " : Le trafic pour tous les protocoles décodés par Snort est résumé dans la section "breakdown by protocol" .
* Actions, Limits, et  Verdicts : Le nombre d'actions et de verdict montre ce que Snort a fait avec les paquets qu'il a analysés. Ces informations ne sont sorties qu'en mode IDS ."Limits " surviennent en raison de contraintes réelles sur le temps de traitement et la mémoire disponible.

```
===============================================================================
Run time for packet processing was 17.7775 seconds
Snort processed 682 packets.
Snort ran for 0 days 0 hours 0 minutes 17 seconds
   Pkts/sec:           40
===============================================================================
Memory usage summary:
  Total non-mmapped bytes (arena):       2297856
  Bytes in mapped regions (hblkhd):      17252352
  Total allocated space (uordblks):      2072368
  Total free space (fordblks):           225488
  Topmost releasable block (keepcost):   69168
===============================================================================
Packet I/O Totals:
   Received:          720
   Analyzed:          682 ( 94.722%)
    Dropped:            0 (  0.000%)
   Filtered:            0 (  0.000%)
Outstanding:           38 (  5.278%)
   Injected:            0
===============================================================================
Breakdown by protocol (includes rebuilt packets):
        Eth:          682 (100.000%)
       VLAN:            0 (  0.000%)
        IP4:          682 (100.000%)
       Frag:            0 (  0.000%)
       ICMP:            0 (  0.000%)
        UDP:          328 ( 48.094%)
        TCP:          347 ( 50.880%)
        IP6:            0 (  0.000%)
    IP6 Ext:            0 (  0.000%)
   IP6 Opts:            0 (  0.000%)
      Frag6:            0 (  0.000%)
      ICMP6:            0 (  0.000%)
       UDP6:            0 (  0.000%)
       TCP6:            0 (  0.000%)
     Teredo:            0 (  0.000%)
    ICMP-IP:            0 (  0.000%)
    IP4/IP4:            0 (  0.000%)
    IP4/IP6:            0 (  0.000%)
    IP6/IP4:            0 (  0.000%)
    IP6/IP6:            0 (  0.000%)
        GRE:            0 (  0.000%)
    GRE Eth:            0 (  0.000%)
   GRE VLAN:            0 (  0.000%)
    GRE IP4:            0 (  0.000%)
    GRE IP6:            0 (  0.000%)
GRE IP6 Ext:            0 (  0.000%)
   GRE PPTP:            0 (  0.000%)
    GRE ARP:            0 (  0.000%)
    GRE IPX:            0 (  0.000%)
   GRE Loop:            0 (  0.000%)
       MPLS:            0 (  0.000%)
        ARP:            0 (  0.000%)
        IPX:            0 (  0.000%)
   Eth Loop:            0 (  0.000%)
   Eth Disc:            0 (  0.000%)
   IP4 Disc:            7 (  1.026%)
   IP6 Disc:            0 (  0.000%)
   TCP Disc:            0 (  0.000%)
   UDP Disc:            0 (  0.000%)
  ICMP Disc:            0 (  0.000%)
All Discard:            7 (  1.026%)
      Other:            0 (  0.000%)
Bad Chk Sum:          335 ( 49.120%)
    Bad TTL:            0 (  0.000%)
     S5 G 1:            0 (  0.000%)
     S5 G 2:            0 (  0.000%)
      Total:          682
===============================================================================
Action Stats:
     Alerts:            0 (  0.000%)
     Logged:            0 (  0.000%)
     Passed:            0 (  0.000%)
Limits:
      Match:            0
      Queue:            0
        Log:            0
      Event:            0
      Alert:            0
Verdicts:
      Allow:          682 ( 94.722%)
      Block:            0 (  0.000%)
    Replace:            0 (  0.000%)
  Whitelist:            0 (  0.000%)
  Blacklist:            0 (  0.000%)
     Ignore:            0 (  0.000%)
      Retry:            0 (  0.000%)
===============================================================================
Snort exiting

```
Aller au répertoire /var/log/snort. Ouvrir le fichier `alert`. Vérifier qu'il y ait des alertes pour votre nom ou mot choisi.

**Question 7: A quoi ressemble l'alerte ? Qu'est-ce que chaque élément de l'alerte veut dire ? Décrivez-la en détail !**

---

**Reponse :**  

---
Lorsque Snort génère un message d'alerte, il se présente généralement comme suit (exemple précédent): 
```
[**] [1:4000122:2] walid find a disney movie! [**]
[Priority: 0] 
04/09-14:59:23.126925 23.236.60.174:80 -> 10.0.2.15:39112
TCP TTL:64 TOS:0x0 ID:49184 IpLen:20 DgmLen:1460
***AP*** Seq: 0x12D92B5E  Ack: 0xF362C3D9  Win: 0xFFFF  TcpLen: 20
```
* Le premier numéro est l'ID du générateur, il indique à l'utilisateur quel composant de Snort a généré cette alerte ainsi le message de l'alerte.
* la case periority indique la periorité de l'alerte générée (par défaut est égale à zéro).
* la troisième ligne indique des informatioms concernant le log et la date de creation .
* à la fin de la sortie de l'alerte on toouve tous les informations conceranant le paquet (protocole ,TTL,MTU ...).
---

### Detecter une visite à Wikipedia

Ecrire une règle qui journalise (sans alerter) un message à chaque fois que Wikipedia est visité **DEPUIS VOTRE** station. **Ne pas utiliser une règle qui détecte un string ou du contenu**.

**Question 8: Quelle est votre règle ? Où le message a-t'il été journalisé ? Qu'est-ce qui a été journalisé ?**

---

**Reponse :**  

---
Règle utilisé :
```
log tcp [2a02:1205:34c1:27c0:a55b:b5f:f517:1c74]  any -> [2620:0:862:ed1a::3]  any (msg: "walid visited Wikipedia"; sid: 40000143; rev:1;)

```
Notre message est journalisé dans un fichiers log  en format pcap dans le dossier /var/log/snort/.c'est une capturre des paquets capturées on peut l'ouvrir avec wireshark ou bien avec la commande tcpdump.

--

### Detecter un ping d'un autre système

Ecrire une règle qui alerte à chaque fois que votre système reçoit un ping depuis une autre machine (je sais que la situation actuelle du Covid-19 ne vous permet pas de vous mettre ensemble... utilisez votre imagination pour trouver la solution à cette question !). Assurez-vous que **ça n'alerte pas** quand c'est vous qui envoyez le ping vers un autre système !

**Question 9: Quelle est votre règle ?**

---

**Reponse :**  

---
```
alert icmp any any -> 192.168.1.107 any (msg:"receiving ping "; itype:8; sid:40000145; rev:1)
````

**Question 10: Comment avez-vous fait pour que ça identifie seulement les pings entrants ?**

---

**Reponse :**  

---
Afin de détecter seulement les ping entrants on a identifier les paquets de protocole ICMP de type 8 (echo request).en précisant aussi que la destination est notre machine (notre adresse ip ).

**Question 11: Où le message a-t-il été journalisé ?**

---

**Reponse :**  

---
Snort enregistre les messages des alertes  dans le fichier /var/log/snort/alert ainsi les paquets capturées qui ont causé l'alerte se trouve dans le log.xxx.

**Question 12: Qu'est-ce qui a été journalisé ?**

---

**Reponse :**  

---
Un message indiquant le paquet entrante ainsi l'identifiant de l'alerte et le ficher log contenant le paquet complèt .

---

### Detecter les ping dans les deux sens

Modifier votre règle pour que les pings soient détectés dans les deux sens.

**Question 13: Qu'est-ce que vous avez modifié pour que la règle détecte maintenant le trafic dans les deux senses ?**

---

**Reponse :**  

---
Voici la nouvelle règle :
```
alert icmp any any <> 192.168.1.107 any (msg:"receiving ping "; sid:40000145; rev:1)
````
on a modifié le sens des paquets de -> vers <>(deux sens ) aussi on a retiré l'option itype:8 pour permettre de détecter les pings sortants aussi .
 
---

### Detecter une tentative de login SSH

Essayer d'écrire une règle qui Alerte qu'une tentative de session SSH a été faite depuis la machine d'un voisin (je sais que la situation actuelle du Covid-19 ne vous permet pas de vous mettre ensemble... utilisez votre imagination pour trouver la solution à cette question !). Si vous avez besoin de plus d'information sur ce qui décrit cette tentative (adresses, ports, protocoles), servez-vous de Wireshark pour analyser les échanges lors de la requête de connexion depuis votre voisi.

**Question 14: Quelle est votre règle ? Montrer la règle et expliquer en détail comment elle fonctionne.**

---

**Reponse :**  

---
```
alert tcp any any -> 192.168.1.107  22 (msg:"SSH login"; sid:100000186; rev:1)

```
La règle établie détecte toutes tentative de connection shh vers notre machine on utilisant le port 22 comme référence .
tous les alertes générées seront enregistrées dans le fichier alertes avec un id unique .

**Question 15: Montrer le message d'alerte enregistré dans le fichier d'alertes.** 

---

**Reponse :**  

---
pas de test effectué eu niveau des connection ssh parce que c'est difficile sans une autre machine malgré que j'ai essayé de me connecter sur des serveurs avec des session ssh .

--- 

### Analyse de logs

Lancer Wireshark et faire une capture du trafic sur l'interface connectée au bridge. Générez du trafic avec votre machine hôte qui corresponde à l'une des règles que vous avez ajouté à votre fichier de configuration personnel. Arrêtez la capture et enregistrez-la dans un fichier.

**Question 16: Quelle est l'option de Snort qui permet d'analyser un fichier pcap ou un fichier log ?**

---

**Reponse :**  

---
Pour analyser le pichier pcap généré par wireshark on peut utiliser l'option : snort -r file.pcap


Utiliser l'option correcte de Snort pour analyser le fichier de capture Wireshark.
```
root@kali:~# sudo snort -r walid.pcapng 
Running in packet dump mode

        --== Initializing Snort ==--
Initializing Output Plugins!
pcap DAQ configured to read-file.
Acquiring network traffic from "walid.pcapng".

        --== Initialization Complete ==--

   ,,_     -*> Snort! <*-
  o"  )~   Version 2.9.7.0 GRE (Build 149) 
   ''''    By Martin Roesch & The Snort Team: http://www.snort.org/contact#team
           Copyright (C) 2014 Cisco and/or its affiliates. All rights reserved.
           Copyright (C) 1998-2013 Sourcefire, Inc., et al.
           Using libpcap version 1.8.1
           Using PCRE version: 8.39 2016-06-14
           Using ZLIB version: 1.2.11

Commencing packet processing (pid=5910)
WARNING: No preprocessors configured for policy 0.
04/09-17:18:58.930462 192.168.1.117:59094 -> 239.255.255.250:1900
UDP TTL:1 TOS:0x0 ID:64060 IpLen:20 DgmLen:202
Len: 174
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

04/09-17:18:59.755383 192.168.1.140:37402 -> 34.246.158.160:443
TCP TTL:64 TOS:0x0 ID:61678 IpLen:20 DgmLen:132 DF
***AP*** Seq: 0x57F85352  Ack: 0xC887E79B  Win: 0x16D  TcpLen: 32
TCP Options (3) => NOP NOP TS: 2397176808 548962748 
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:18:59.836741 34.246.158.160:443 -> 192.168.1.140:37402
TCP TTL:224 TOS:0x0 ID:45221 IpLen:20 DgmLen:52 DF
***A**** Seq: 0xC887E79B  Ack: 0x57F853A2  Win: 0x74  TcpLen: 32
TCP Options (3) => NOP NOP TS: 548965555 2397176808 
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:18:59.852428 192.168.1.117:59094 -> 239.255.255.250:1900
UDP TTL:1 TOS:0x0 ID:64061 IpLen:20 DgmLen:202
Len: 174
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:18:59.987079 52.114.76.163:443 -> 192.168.1.107:54376
TCP TTL:109 TOS:0x0 ID:50397 IpLen:20 DgmLen:169 DF
***AP*** Seq: 0xBA57D457  Ack: 0x57EEA411  Win: 0x802  TcpLen: 20
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:18:59.987101 52.114.76.163:443 -> 192.168.1.107:54376
TCP TTL:109 TOS:0x0 ID:50398 IpLen:20 DgmLen:250 DF
***AP*** Seq: 0xBA57D4D8  Ack: 0x57EEA411  Win: 0x802  TcpLen: 20
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:18:59.987105 52.114.76.163:443 -> 192.168.1.107:54376
TCP TTL:109 TOS:0x0 ID:50399 IpLen:20 DgmLen:86 DF
***AP*** Seq: 0xBA57D5AA  Ack: 0x57EEA411  Win: 0x802  TcpLen: 20
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:18:59.987109 52.114.76.163:443 -> 192.168.1.107:54376
TCP TTL:109 TOS:0x0 ID:50400 IpLen:20 DgmLen:78 DF
***AP*** Seq: 0xBA57D5D8  Ack: 0x57EEA411  Win: 0x802  TcpLen: 20
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:18:59.987113 192.168.1.107:54376 -> 52.114.76.163:443
TCP TTL:128 TOS:0x0 ID:24756 IpLen:20 DgmLen:40 DF
***A**** Seq: 0x57EEA411  Ack: 0xBA57D5FE  Win: 0x205  TcpLen: 20
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:19:00.003281 192.168.1.107:54376 -> 52.114.76.163:443
TCP TTL:128 TOS:0x0 ID:24757 IpLen:20 DgmLen:224 DF
***AP*** Seq: 0x57EEA411  Ack: 0xBA57D5FE  Win: 0x205  TcpLen: 20
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:19:00.083773 52.114.76.163:443 -> 192.168.1.107:54376
TCP TTL:109 TOS:0x0 ID:50401 IpLen:20 DgmLen:40 DF
***A**** Seq: 0xBA57D5FE  Ack: 0x57EEA4C9  Win: 0x801  TcpLen: 20
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:19:00.203956 140.82.114.25:443 -> 192.168.1.107:56914
TCP TTL:48 TOS:0x0 ID:273 IpLen:20 DgmLen:64 DF
***AP*** Seq: 0x4327CAA  Ack: 0x313ACEC7  Win: 0x20  TcpLen: 20
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:19:00.204126 192.168.1.107:56914 -> 140.82.114.25:443
TCP TTL:128 TOS:0x0 ID:16602 IpLen:20 DgmLen:68 DF
***AP*** Seq: 0x313ACEC7  Ack: 0x4327CC2  Win: 0x1FF  TcpLen: 20
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:19:00.300903 140.82.114.25:443 -> 192.168.1.107:56914
TCP TTL:48 TOS:0x0 ID:274 IpLen:20 DgmLen:40 DF
***A**** Seq: 0x4327CC2  Ack: 0x313ACEE3  Win: 0x20  TcpLen: 20
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:19:00.418607 192.168.1.107:56544 -> 198.252.206.25:443
TCP TTL:128 TOS:0x0 ID:30099 IpLen:20 DgmLen:41 DF
***A**** Seq: 0xF2B4A7B2  Ack: 0xBAAC704D  Win: 0x1FD  TcpLen: 20
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:19:00.518131 198.252.206.25:443 -> 192.168.1.107:56544
TCP TTL:48 TOS:0x0 ID:1301 IpLen:20 DgmLen:52 DF
***A**** Seq: 0xBAAC704D  Ack: 0xF2B4A7B3  Win: 0x3F  TcpLen: 32
TCP Options (3) => NOP NOP Sack: 62132@42930 
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:19:01.083490 192.168.1.117:59094 -> 239.255.255.250:1900
UDP TTL:1 TOS:0x0 ID:64062 IpLen:20 DgmLen:202
Len: 174
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:19:02.002480 192.168.1.117:59094 -> 239.255.255.250:1900
UDP TTL:1 TOS:0x0 ID:64063 IpLen:20 DgmLen:202
Len: 174
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:19:02.287188 192.168.1.107:56882 -> 140.82.113.25:443
TCP TTL:128 TOS:0x0 ID:25782 IpLen:20 DgmLen:41 DF
***A**** Seq: 0xF7AB38D1  Ack: 0xAE056174  Win: 0x1FE  TcpLen: 20
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:19:02.384244 140.82.113.25:443 -> 192.168.1.107:56882
TCP TTL:53 TOS:0x20 ID:38086 IpLen:20 DgmLen:52 DF
***A**** Seq: 0xAE056174  Ack: 0xF7AB38D2  Win: 0x20  TcpLen: 32
TCP Options (3) => NOP NOP Sack: 63403@14545 
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:19:02.994683 192.168.1.107:54203 -> 64.225.7.146:443
TCP TTL:128 TOS:0x0 ID:48388 IpLen:20 DgmLen:41 DF
***A**** Seq: 0x5FB91682  Ack: 0x2F7AEBDF  Win: 0x200  TcpLen: 20
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/09-17:19:03.083414 64.225.7.146:443 -> 192.168.1.107:54203
TCP TTL:53 TOS:0x0 ID:22790 IpLen:20 DgmLen:52 DF
***A**** Seq: 0x2F7AEBDF  Ack: 0x5FB91683  Win: 0x10F  TcpLen: 32
TCP Options (3) => NOP NOP Sack: 24505@5762 
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

===============================================================================
Run time for packet processing was 0.254 seconds
Snort processed 22 packets.
Snort ran for 0 days 0 hours 0 minutes 0 seconds
   Pkts/sec:           22
===============================================================================
Memory usage summary:
  Total non-mmapped bytes (arena):       786432
  Bytes in mapped regions (hblkhd):      12906496
  Total allocated space (uordblks):      684736
  Total free space (fordblks):           101696
  Topmost releasable block (keepcost):   84464
===============================================================================
Packet I/O Totals:
   Received:           22
   Analyzed:           22 (100.000%)
    Dropped:            0 (  0.000%)
   Filtered:            0 (  0.000%)
Outstanding:            0 (  0.000%)
   Injected:            0
===============================================================================
Breakdown by protocol (includes rebuilt packets):
        Eth:           22 (100.000%)
       VLAN:            0 (  0.000%)
        IP4:           22 (100.000%)
       Frag:            0 (  0.000%)
       ICMP:            0 (  0.000%)
        UDP:            4 ( 18.182%)
        TCP:           18 ( 81.818%)
        IP6:            0 (  0.000%)
    IP6 Ext:            0 (  0.000%)
   IP6 Opts:            0 (  0.000%)
      Frag6:            0 (  0.000%)
      ICMP6:            0 (  0.000%)
       UDP6:            0 (  0.000%)
       TCP6:            0 (  0.000%)
     Teredo:            0 (  0.000%)
    ICMP-IP:            0 (  0.000%)
    IP4/IP4:            0 (  0.000%)
    IP4/IP6:            0 (  0.000%)
    IP6/IP4:            0 (  0.000%)
    IP6/IP6:            0 (  0.000%)
        GRE:            0 (  0.000%)
    GRE Eth:            0 (  0.000%)
   GRE VLAN:            0 (  0.000%)
    GRE IP4:            0 (  0.000%)
    GRE IP6:            0 (  0.000%)
GRE IP6 Ext:            0 (  0.000%)
   GRE PPTP:            0 (  0.000%)
    GRE ARP:            0 (  0.000%)
    GRE IPX:            0 (  0.000%)
   GRE Loop:            0 (  0.000%)
       MPLS:            0 (  0.000%)
        ARP:            0 (  0.000%)
        IPX:            0 (  0.000%)
   Eth Loop:            0 (  0.000%)
   Eth Disc:            0 (  0.000%)
   IP4 Disc:            0 (  0.000%)
   IP6 Disc:            0 (  0.000%)
   TCP Disc:            0 (  0.000%)
   UDP Disc:            0 (  0.000%)
  ICMP Disc:            0 (  0.000%)
All Discard:            0 (  0.000%)
      Other:            0 (  0.000%)
Bad Chk Sum:            1 (  4.545%)
    Bad TTL:            0 (  0.000%)
     S5 G 1:            0 (  0.000%)
     S5 G 2:            0 (  0.000%)
      Total:           22
===============================================================================
Snort exiting

```

**Question 17: Quelle est le comportement de Snort avec un fichier de capture ? Y-a-t'il une difference par rapport à l'analyse en temps réel ?**

---

**Reponse :**  

---

Selon la documentation officielle du snort ,ce dernier analyse le ficher selon sa propre liste des règles configurés donc l'analyse n'est pas en temps réel .

**Question 18: Est-ce que des alertes sont aussi enregistrées dans le fichier d'alertes?**

---

**Reponse :**  

---
Oui parce que snort analyse les paquets dans le fichier comme c'était en écoute 

--

### Contournement de la détection

Faire des recherches à propos des outils `fragroute` et `fragtest`.

**Question 20: A quoi servent ces deux outils ?**

---

**Reponse :**  

---
il servent à intercepter, modifier et réécrire  le trafic de sortie destiné à un hôte spécifié.


**Question 21: Quel est le principe de fonctionnement ?**

---

**Reponse :**  

---
ces outils modifie les paquets afin d'eviter la détection par les règles mises en place

**Question 22: Qu'est-ce que le `Frag3 Preprocessor` ? A quoi ça sert et comment ça fonctionne ?**

---

**Reponse :**  

---
Le préprocesseur frag3 est un module de défragmentation IP basé sur cible pour Snort. Frag3 est conçu avec les objectifs suivants: Une exécution plus rapide avec une gestion des données moins complexe.Techniques anti-évasion basées sur la cible de modélisation d'hôte.
source :https://www.snort.org/faq/readme-frag3 

Reprendre l'exercice de la partie [Trouver votre nom](#trouver-votre-nom-). Essayer d'offusquer la détection avec `fragroute`.


**Question 23: Quel est le résultat de votre tentative ?**

---

**Reponse :**  

---
tous les testes marchent trés bien mais une erreur est enlevée au niveau des régles parce que le frag3 preprocesseur n'est pas connu.

Modifier le fichier `myrules.rules` pour que snort utiliser le `Frag3 Preprocessor` et refaire la tentative.


**Question 24: Quel est le résultat ?**

---

**Reponse :**  

---
Grace à frag3 snort ne détecte pas les alerte. le module contourne les règles avec des modifications au niveau des paquets .


**Question 25: A quoi sert le `SSL/TLS Preprocessor` ?**

---

**Reponse :**  

---
Chaque paquet contenant du trafic SSL a une partie non chiffrée qui fournit des informations sur le trafic lui-même et l'état de la connexion.le SSL/TLS Preprocessor sert à analyser ces parties non chiffrées .

**Question 26: A quoi sert le `Sensitive Data Preprocessor` ?**

---

**Reponse :**  

---
le `Sensitive Data Preprocessor `est un module Snort qui effectue la détection et le filtrage des informations personnelles identifiables.
source :https://www.snort.org/faq/readme-sensitive_data


### Conclusions


**Question 27: Donnez-nous vos conclusions et votre opinion à propos de snort**

---

**Reponse :**  

---
Snort c'est un excellent outils de détection qui contient une varité  des options nous permet de manipuler et sécuriser notre réseau .


<sub>This guide draws heavily on http://cs.mvnu.edu/twiki/bin/view/Main/CisLab82014</sub>
