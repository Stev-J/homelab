# Lab 01 – Infrastructure PME multi-VLAN avec pare-feu et DMZ

Conception et mise en œuvre d'un réseau d'entreprise complet segmentation par service,
routage inter-VLAN, adressage dynamique, pare-feu périmétrique et zone démilitarisée
hébergeant un serveur web accessible depuis Internet.

Réalisé sous Cisco Packet Tracer. Cette infrastructure a été conçue dans le cadre de ma
préparation au titre professionnel TSSR, puis présentée et validée par le service réseaux
d'une collectivité territoriale.

## Contexte

Une PME d'une soixantaine de postes répartis sur cinq services. Les besoins :

- isoler les services les uns des autres (un poste Accueil ne doit pas atteindre les serveurs)
- centraliser l'attribution des adresses pour le parc informatique
- exposer un serveur web sur Internet sans ouvrir le réseau interne
- garder une sortie Internet contrôlée pour tous les services

## Topologie

<img width="2505" height="1206" alt="Topologie du réseau sous Packet Tracer" src="https://github.com/user-attachments/assets/51fdd769-6d4e-4198-b8a7-452b03fe1f67" />

| VLAN | Service     | Réseau         | Adressage |
|------|-------------|----------------|-----------|
| 10   | Informatique| 10.0.10.0/24   | DHCP (pool 10.0.10.20 et suivantes) |
| 20   | Comptabilité| 10.0.20.0/24   | Statique  |
| 30   | Direction   | 10.0.30.0/24   | Statique  |
| 40   | RH          | 10.0.40.0/24   | Statique  |
| 50   | Accueil     | 10.0.50.0/24   | Statique  |

Passerelle de chaque VLAN en .254 sur le switch de niveau 3.

Liens de transit en /30 (deux adresses utiles, pas de gaspillage) :

| Lien | Réseau | Extrémité A | Extrémité B |
|------|--------|-------------|-------------|
| Pare-feu ↔ Router1 | 10.0.0.0/30 | FireWall Gi1/2 — 10.0.0.2 | Router1 Fa0/1 — 10.0.0.1 |
| Router1 ↔ Switch L3 | 10.0.0.4/30 | Router1 Fa0/0 — 10.0.0.6 | SW-L3 Gi1/0/23 — 10.0.0.5 |
| Pare-feu ↔ Routeur FAI | 203.0.113.0/30 | FireWall Gi1/1 — 203.0.113.2 | Routeur_FAI Gig0/0 — 203.0.113.1 |

DMZ : 172.16.50.0/24, serveur web en 172.16.50.10

## Choix de conception

**Pourquoi un switch de niveau 3 plutôt qu'un router-on-a-stick**
Le routage inter-VLAN est assuré en commutation, donc en matériel. Sur un réseau de cette
taille, tout le trafic entre services n'a pas à remonter jusqu'au routeur ni à partager
un seul lien trunk.

**Pourquoi le routeur FAI n'a aucune route vers les réseaux internes**
Il ne connaît que ses réseaux directement connectés. Le NAT du pare-feu traduit toutes les
adresses privées en 203.0.113.2, donc le FAI répond toujours à une adresse qu'il connaît.
C'est le fonctionnement réel d'un accès opérateur le FAI n'a pas à connaître le plan
d'adressage privé de son client.

**Pourquoi des /30 sur les liens de transit**
Un lien point à point n'a besoin que de deux adresses. Un /24 sur un transit, c'est 252
adresses perdues et un plan d'adressage illisible.

**Pourquoi une DMZ séparée**
Le serveur web est joignable depuis Internet, donc c'est la machine la plus exposée du
réseau. S'il est compromis, l'attaquant se retrouve dans une zone cloisonnée, pas sur le
même réseau que la comptabilité. Sur l'ASA, les niveaux de sécurité formalisent ça :
inside à 100 (confiance totale), dmz à 50, outside à 0 (aucune confiance).

**Pourquoi un DHCP centralisé avec relais**
Serveur DHCP dédié en 10.0.10.10, un pool pour le VLAN 10 : passerelle 10.0.10.254,
DNS 8.8.8.8, distribution à partir de 10.0.10.20 pour laisser le début de plage aux
équipements fixes. Les interfaces VLAN 20 à 50 du switch L3 portent un
`ip helper-address 10.0.10.10` : une requête DHCP est un broadcast, qui ne franchit pas
les frontières de VLAN. Le relais transforme ce broadcast en requête unicast vers le
serveur, ce qui évite d'avoir un serveur DHCP par service.

**Pourquoi deux switches d'accès par service**
Répartition de la charge par port et possibilité d'intervenir sur un switch sans couper
tout un service.

## Mise en œuvre

### Pare-feu (Cisco ASA)

Nommage des interfaces et affectation des niveaux de sécurité :

```
hostname Firewall

interface GigabitEthernet1/1
 description Liaison_Vers_Internet_FAI
 nameif outside
 security-level 0
 ip address 203.0.113.2 255.255.255.252

interface GigabitEthernet1/2
 description Liaison_Vers_Reseau_Interne
 nameif inside
 security-level 100
 ip address 10.0.0.2 255.255.255.252

interface GigabitEthernet1/3
 description Liaison_Vers_Serveur_Web
 nameif dmz
 security-level 50
 ip address 172.16.50.254 255.255.255.0
```

Traduction d'adresses pour la sortie Internet :

```
object network TOUS_LES_RESEAUX
 subnet 0.0.0.0 0.0.0.0
 nat (inside,outside) dynamic interface
```

Un seul objet couvrant toutes les adresses internes, traduites vers l'adresse publique de
l'interface outside.

Inspection ICMP — sans elle, les réponses aux pings sortants sont bloquées au retour :

```
policy-map global_policy
 class inspection_default
  inspect icmp
```

Configuration complète : [firewall-asa.txt](./configs/firewall-asa.txt)

### Switch de niveau 3

Déclaration des VLAN, interfaces virtuelles avec les passerelles en .254, relais DHCP,
liaison routée vers le routeur et route par défaut.
Configuration complète : [sw-l3.txt](./configs/sw-l3.txt)

### Routeur

Placé entre le pare-feu et le switch de niveau 3. Il porte la route par défaut vers le
pare-feu et une route de retour vers les réseaux internes.
Configuration complète : [routeur.txt](./configs/routeur.txt)

### Routeur FAI et switch d'accès

[routeur-fai.txt](./configs/routeur-fai.txt) · [sw-vlan10.txt](./configs/sw-vlan10.txt)

## Fichier Packet Tracer

La topologie complète est téléchargeable : [infra-multivlan-dmz.pkt](./infra-multivlan-dmz.pkt)

Ouvrir avec Cisco Packet Tracer 8.x (gratuit avec un compte Cisco NetAcad).

## Tests de recette

| # | Test | Résultat attendu | Statut |
|---|------|------------------|--------|
| 1 | Poste VLAN 10 obtient une IP par DHCP | Bail à partir de 10.0.10.20 | OK |
| 2 | Ping entre deux postes du même VLAN | Réponse | OK |
| 3 | Ping inter-VLAN 50 vers 10 | Réponse via le switch L3 (1er paquet perdu résolution ARP) | OK |
| 4 | Ping poste interne vers 8.8.8.8 | Réponse via NAT (1er paquet perdu résolution ARP) | OK |
| 5 | `show vlan brief` sur le switch L3 | VLAN 10/20/30/40/50 actifs | OK |

### Captures

**1. Poste du VLAN 10 ayant obtenu son adresse par DHCP**

<img width="847" height="287" alt="Poste du VLAN 10 ayant obtenu son adresse par DHCP" src="https://github.com/user-attachments/assets/8ed1189f-4bdb-4a10-970d-936388128350" />


**2. Ping entre deux postes du même VLAN**

<img width="583" height="391" alt="Ping réussi entre deux postes du même VLAN" src="https://github.com/user-attachments/assets/0cca5722-5100-4882-acbb-473ab513a484" />


**3. Ping inter-VLAN du 50 vers le 10**

<img width="858" height="399" alt="Ping inter-VLAN du 50 vers le 10" src="https://github.com/user-attachments/assets/a75c9055-a5e2-4acb-b4e0-617d1c717426" />


**4. Ping d'un poste interne vers 8.8.8.8**

<img width="586" height="371" alt="Ping vers 8.8.8.8 à travers le NAT du pare-feu" src="https://github.com/user-attachments/assets/fd2c8f3a-0641-4316-b89e-48cd00984de1" />


**5. `show vlan brief` sur le switch de niveau 3**

<img width="837" height="739" alt="Sortie de show vlan brief sur le switch de niveau 3" src="https://github.com/user-attachments/assets/abf78f64-beb4-421c-bb42-8aed34c7c162" />

## Problèmes rencontrés

Cette section rassemble les défauts relevés en reprenant la maquette et en relisant les
configurations ligne à ligne, plusieurs mois après sa conception. Chacun est décrit sous
la forme : symptôme, vérification, cause, correction.

### Conflit d'adresse IP entre le switch d'accès et la passerelle

Symptôme : connectivité instable sur le VLAN 10, sans schéma de reproduction évident.
Vérification : `show running-config` sur les deux équipements. L'interface `Vlan10` du
switch d'accès SW-VLAN10 et celle du switch de niveau 3 portaient toutes deux
10.0.10.254.
Cause : deux équipements avec la même adresse sur le même réseau. Selon celui qui répond
le premier aux requêtes ARP, un poste peut envoyer son trafic au switch d'accès, qui ne
sait pas router hors du VLAN.
Correction : sur un switch d'accès, l'interface VLAN ne sert qu'à l'administration à
distance. Elle a été basculée en 10.0.10.251, hors de la plage DHCP et distincte de la
passerelle.
Ce que j'en retire : une erreur d'adressage ne provoque pas toujours une panne franche.
Un conflit d'IP produit un comportement intermittent, bien plus difficile à diagnostiquer
qu'un lien coupé.

### Les pings sortants ne revenaient pas

Symptôme : depuis un poste interne, aucune réponse à un ping vers Internet, alors que le
NAT était en place et que le pare-feu joignait lui-même 8.8.8.8.
Cause : l'ASA bloque par défaut le trafic de retour ICMP, qui n'est pas inspecté.
Correction : `inspect icmp` dans la policy-map globale.
Ce que j'en retire : sur un pare-feu à états, le sens d'ouverture d'un flux compte autant
que la règle elle-même.

### Relais DHCP configurés sans pool correspondant

Symptôme : les postes des VLAN 20 à 50 n'obtiendraient aucun bail, alors que
`ip helper-address 10.0.10.10` est bien présent sur chaque interface VLAN du switch L3.
Vérification : la console du serveur DHCP ne montre qu'un seul pool, `serverPool`,
couvrant 10.0.10.0/24.
Cause : le relais achemine bien la requête jusqu'au serveur, mais celui-ci n'a aucune
plage à proposer pour les autres sous-réseaux et ne répond donc pas.
Ce que j'en retire : un relais DHCP ne crée pas d'adresses, il transporte une demande.
Les deux moitiés de la chaîne doivent être configurées. Les postes de ces VLAN étant
adressés en statique dans la maquette, le défaut restait invisible.
Correction envisagée : ajouter un pool par VLAN sur le serveur, ou retirer les relais
tant qu'ils ne servent à rien.

### Description d'interface issue d'une commande mal placée

Symptôme : la description du port trunk du switch d'accès affichait `write memory`.
Cause : la commande de sauvegarde avait été saisie alors que la console était encore en
mode configuration d'interface. L'IOS l'a interprétée comme la suite d'une commande
`description`, et la configuration n'a pas été sauvegardée à ce moment-là.
Correction : description remplacée par `Trunk_Vers_SW-L3`, sauvegarde refaite depuis le
mode EXEC privilégié.
Ce que j'en retire : vérifier le niveau d'invite avant de valider une commande, et
relire le résultat d'un `show running-config` plutôt que de supposer que la saisie est
passée.

### Objets NAT redondants sur le pare-feu

Constat : deux objets réseau déclarés avec la même règle de traduction, `RESEAU_GLOBAL`
(10.0.0.0/8) et `TOUS_LES_RESEAUX` (0.0.0.0/0), tous deux en `nat (inside,outside)
dynamic interface`. Le second englobant le premier, un seul suffit.
Ce que j'en retire : une configuration qui fonctionne n'est pas pour autant propre. Les
règles superflues compliquent la lecture et les futurs dépannages.

## Ce que je ferais différemment aujourd'hui

- **Aucune redondance** : la panne du switch L3 ou du pare-feu coupe tout. Un second lien
  avec HSRP et du spanning-tree correctement dimensionné changerait ça.
- **VLAN natif laissé en 1** sur les trunks, ce qui expose au VLAN hopping. Je le
  basculerais sur un VLAN dédié et inutilisé.
- **Tous les ports du switch L3 en trunk**, y compris ceux qui ne sont reliés à rien,
  comme le montre la sortie de `show vlan brief`. Un port trunk libre est une porte
  ouverte : en production, les ports inutilisés passent en access sur un VLAN mort et
  sont désactivés (`shutdown`).
- **Pools DHCP incomplets** : les relais sont en place sur quatre VLAN mais le serveur
  n'héberge qu'un pool. À compléter ou à retirer.
- **Pas de VLAN d'administration séparé** : les équipements sont joignables depuis les
  réseaux métier. Un VLAN 99 réservé à l'administration, accessible depuis le seul poste
  technicien, serait plus sain.
- **Pas de supervision** : aucun moyen de savoir qu'un lien est tombé avant que les
  utilisateurs appellent.
- **Lignes vty en `login` sans mot de passe défini**, ce qui rend l'accès distant
  inopérant. À remplacer par du SSH avec des comptes nominatifs et un
  `transport input ssh`.

## Environnement

Cisco Packet Tracer : switches d'accès 2960, switch de niveau 3, routeurs 1941 et 1841,
pare-feu ASA 5506, serveur DHCP, serveur web, postes clients.
Trunking 802.1Q, relais DHCP, NAT/PAT, routes statiques, policy-map ASA.
