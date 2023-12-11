---
layout: post
title:  "(True)NAS comme à la maison avec Wireguard + pfSense"
date:   2023-12-11 23:36:54 +0100
categories: tutos pfsense truenas wireguard
---
[center]![Wireguard+pfSense ](/assets/images/1701962718-221022-image.png)
 ## Et une couche de TrueNAS Scale[/center]

Sur un coup de tête, en période de "vacances", avec les copains dans le casque on dit beaucoup de conneries et nous en faisons tout autant. 
Profitant que @"Greg"#8477 ait voulu basculer de unRAID pour un TrueNAS ( on veut seulement la partie NAS ), j'ai re découvert ce dernier et pas mal d'évolutions.

Enfin tout ça pour que dans mon crâne, je décide de profiter d'un serveur à dispo pour ressortir mon truenas de son coldstorage et refaire un truenas.

Problématique TrueNAS est sur un serveur Hetzner 😅donc tu le vois venir je ne vais pas ouvrir un protocole SMB-CIFS/NFS sur le net, même avec du firewall hein ! 
Et puis monopoliser une ipv4 publique comme ça en ce moment juste pour un truc simple n'est pas à l'ordre du jour... Par contre j'ai quelques IPv6 encore de dispos dans mon pool xD

Résultat : installer pfSense en frontale, mais avec seulement une ipv6 publique et mettre le truenas sur une patte lan mais donc "protéger" d'internet. Et pour accéder à tout ça on va profiter de la partie WireGuard pour créer un tunnel.

La partie que j'utilise c'est le split-tunneling, juste faire passer le flux nécessaire pour atteindre le TrueNAS. Pas avoir une ip en sortie de chez Hetzner, pour du full-vpn tunnel j'utilise ma stack adWireGuard sur le pi4 à la maison quand je suis dehors, ou sur celle de Oracle enfin chaque utilisation a son environnement ;)
[center]![schéma réseau](/assets/images/1701962738-455275-image.png)[/center]
[center]_Image tirée du site : https://www.laroberto.com_[/center]
[center]Sur cette image, il faut pense que Server = pfSense, et Router = TrueNAS.[/center]
> ( Pour mémoire, j'utilise une partie de son article d'où est tirée cette image pour une autre utilisation dans un cadre professionnel afin de ne pas avoir à ouvrir de port du côté du lan entreprise )

### Résumons : 
- J'ai besoin d'accéder à un truenas pour le stockage réseau. 
- Je n'ai pas envie de mettre une ipv4 publique, et d'avoir un flux en clair qui passe.
- J'avais envie de découvrir wireguard sur pfsense et de m'y essayer.
- Et puis c'était l'occasion de faire des conneries avec les copains !

### Etape 1 : avoir un pfsense qui tourne 😅
Installer wireguard donc sur ce dernier une fois installé on le retrouve dans l'onglet VPN ( la partie installation se passe dans System -> Package Manager )
[center]![wireguard pfsense](/assets/images/1701962767-974820-image.png)[/center]

### Etape 2 : configurer le tunnel et le peer
[center]![tunnel wireguard](/assets/images/1701962775-261594-image.png)[/center]
**Enable** : coché ( on l'active )
**Description** : Remote Access ( faut bien qu'on sache à quoi qui sert 😅)
**Listen Port** : 51821 ici mais le port par défaut c'est 51820/udp en gros tu choisis le port sur lequel le serveur wireguard écoute pour initier la connexion.
**Interfaces Keys** : on clique sur GENERATE pour créer la paire de clés pour ce tunnel, la partie Public servira dans le fichier de conf sur le client.
**Interface Addresses** : le plan d'adressage dans le tunnel par exemple 10.6.210.1/24 sera l'adresse de pfSense dans le tunnel, le /24 lui permettra de mettre plein de clients. Si par sécurité on veut bloquer on peut remplacer par un /30 ce qui permettra d'utiliser 10.6.210.1 et 10.6.210.2 donc pfsense + 1 client.
Je ne vais pas faire un cours complet sur ce point là.
**Et hop on SAVE !**
Nous voilà avec un tunnel reste à mettre un client.

### Etape 3 : Générer la paire de clé publique/privée côté client.
Sous windows, quand on crée un tunnel dans wireguard il les affiches : 
![windows tunnel ](/assets/images/1701962791-339903-image.png)
Sous linux les commandes suivantes, permettent de générer la paire de clés:
```
$ wg genkey | tee privatekey | wg pubkey > publickey
$ cat privatekey
WGpL3/ejM5L9ngLoAtXkSP1QTNp4eSD34Zh6/Jfni1Q=
$ cat publickey
b9FjbupGC7fomO5U4jL5Irt1ZV5rq4c+utGKj53HXgU=
```
Donc nous voilà avec nos clés on peut aller créer notre peer sur pfsense.

### Etape 4 : Créer un peer
![pfsense wireguard](/assets/images/1701962819-965576-image.png)
on sélectionne l'onglet Peers et on clique sur Add Peers ( le bouton vert en bas à droite 😅)
![pfsense wireguard](/assets/images/1701962829-197022-image.png)
**Enable** : Coché, on va l'activer
**Tunnel** : Tu sélectionnes le tunnel précédemment créer ( tu dois en avoir qu'un )
**Description** : Nom du client ( ex Mathieu )
**Dynamic Endpoint** : Coché mais on peut ( faut que j'explore la piste ) bloquer sur l'arrivée depuis un seul point ce qui rend donc par exemple le client non nomade.
**Public Key** : on colle la clé publique récupérer sur le client ( la clé publique hein )
**Pre-shared Key** : non utilisé ici
**Allowed IPs** : 10.6.210.2/32 <- on prend la première ip après celle qu'on a donnée au serveur wireguard dans le tunnel, si tu fais un autre client ça sera 10.6.210.3 etc etc
**SAVE PEER !!**

### Etape 5 : créer le fichier de conf à utiliser sur le client
```
[Interface]
PrivateKey = CLE PRIVEE RECUPERE SUR LE CLIENT AU DESSUS
Address = 10.6.210.2/32 <- l'adresse qu'on a donner au peer

[Peer]
PublicKey = CLE PUBLIQUE QU'ON RECUPERE SUR LE SERVEUR WIREGUARD (Tunnel -> Interface -> Clé publique
AllowedIPs = 10.6.210.1/32, 192.168.20.0/24 <- Ici on autorise l'acces au pfsense et au lan du pfsense ( le split tunneling qui permet donc d'acceder au ressources du lan, truenas, imprimantes, pabx, router, etc )
Endpoint = IPPUBLIQUEDUPFSENSE:51820
```

### Etape 6 : quelques règles de firewall
On a donc le serveur actif, un client et le fichier de conf dans le client mais il faudrait faire quelques règles de pare-feu quand même ;)
Dans mon cas, je privilégie l'ipv6 dans la majorité des cas. 
J'ai donc attribué une ipv6 (publique) à pfsense et une ipv4 privée sur la patte wan fais un petit tour sur le tuto qui traine sur le forum de quand j'ajoute des ips sur mon proxmox j'ai une partie vmbr1 avec un lan 10.20.30.0/24 qui permet d'avoir une connectivité ipv4 sans pouvoir être jointe depuis l'extérieur.
Donc la règle à ajouter sur **Firewall -> Rules -> WAN** est la suivante :
![pfsense wireguard](/assets/images/1701962874-916830-image.png)
**Action** : Pass ( on l'autorise )
**Interface** : WAN ( on vient de dehors donc on arrive sur WAN )
**Address Family** : IPv6 ( dans mon cas ) mais si tu as un pfsense avec une ipv4 tu peux faire une règle pour ipv4 seulement ou les deux...
**Protocol** : UDP ( wireguard tourne sur le port 51820/UDP par défaut )
**Source** : Any ( mais on peut bloquer à une ip, un range etc )
**Destination** : WAN address ( puisque le endpoint pour se connecter c'est l'adresse de pfSense )
**Destination Port Range : from Other - 51820 to Other 51820
**Log** : c'est un peu comme tu veut, si tu veut logguer les packets qui passe par cette règle ou pas. 
**Description** : Wireguard port ( histoire un jour de pas l'effacer comme un sagouin parce qu'on se rappelle plus du pourquoi et du comment.
**On peut SAVE !**
Il nous en reste une à faire, elle permettra a notre client de communiquer en dehors du tunnel que ca soit pour aller sur le lan ou sur internet selon l'utilisation du tunnel.

**Firewall -> Rules -> WireGuard** et on ajoute cette règle :
![pfsense wireguard](/assets/images/1701962884-557739-image.png)
Je ne vais pas réexpliquer la totalité mais il faut comprendre qu'on peut bloquer certains protocoles, en autoriser d'autre ici etc pour ma part je vais faire un truc en any ;)

### Etape 7 : Tadaaaa !
![pfsense wireguard](/assets/images/1701962892-87304-image.png)
![pfsense wireguard](/assets/images/1701962895-852594-image.png)

Bien sur, ce tuto/mémo est ouvert aux modifications, adaptations, améliorations et critiques constructives 😘
