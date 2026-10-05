# Lab 01 – Infrastructure PME multi-VLAN avec pare-feu et DMZ

Conception et mise en œuvre d'un réseau d'entreprise complet : segmentation par service,
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

<img width="2505" height="1206" alt="image" src="https://github.com/user-attachments/assets/51fdd769-6d4e-4198-b8a7-452b03fe1f67" />


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
| Router1 ↔ Switch L3 | 10.0.0.4/30 | Router1 Fa0/0 — 10.0.0.6 | SW-L3 |
| Pare-feu ↔ Routeur FAI | 203.0.113.0/30 | FireWall Gi1/1 — 203.0.113.2 | Routeur_FAI Gig0/0 — 203.0.113.1 |

DMZ : 172.16.50.0/24, serveur web en 172.16.50.10

## Choix de conception

**Pourquoi un switch de niveau 3 plutôt qu'un router-on-a-stick**
Le routage inter-VLAN est assuré en commutation, donc en matériel. Sur un réseau de cette
taille, tout le trafic entre services n'a pas à remonter jusqu'au routeur ni à partager
un seul lien trunk.

**Pourquoi des /30 sur les liens de transit**
Un lien point à point n'a besoin que de deux adresses. Un /24 sur un transit, c'est 252
adresses perdues et un plan d'adressage illisible.

**Pourquoi une DMZ séparée**
Le serveur web est joignable depuis Internet, donc c'est la machine la plus exposée du
réseau. S'il est compromis, l'attaquant se retrouve dans une zone cloisonnée, pas sur le
même réseau que la comptabilité. Sur l'ASA, les niveaux de sécurité formalisent ça :
inside à 100 (confiance totale), dmz en intermédiaire, outside à 0 (aucune confiance).

**Pourquoi le DHCP sur le seul VLAN Informatique**
Serveur DHCP dédié, un pool servant le VLAN 10 : passerelle 10.0.10.254, DNS 8.8.8.8,
distribution à partir de 10.0.10.20 pour laisser le début de plage aux équipements fixes.
Les autres services restent en statique dans cette maquette ; en production, le même
serveur couvrirait les autres VLAN via des relais DHCP (`ip helper-address`) sur les
interfaces du switch L3.

**Pourquoi deux switches d'accès par service**
Répartition de la charge par port et possibilité d'intervenir sur un switch sans couper
tout un service.

## Mise en œuvre

### Pare-feu (Cisco ASA)

Nommage des interfaces et affectation des niveaux de sécurité :

```
hostname FireWall_DMZ_INTERNET

interface gigabitEthernet1/1
 description Liaison_Vers_Internet_FAI
 nameif outside
 security-level 0
 ip address 203.0.113.2 255.255.255.252
 no shutdown

interface gigabitEthernet1/2
 description Liaison_Vers_Reseau_Interne
 nameif inside
 security-level 100
 ip address 10.0.0.2 255.255.255.252
 no shutdown

interface gigabitEthernet1/3
 description Liaison_Vers_Serveur_Web
 nameif dmz
 security-level 50
 ip address 172.16.50.254 255.255.255.0
 no shutdown
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

### Switch de niveau 3

Déclaration des VLAN, interfaces virtuelles, affectation des ports d'accès et activation
du routage. Configuration complète : [sw-l3.txt](./configs/sw-l3.txt)

### Routeur

Placé entre le pare-feu et le switch de niveau 3. Il porte la route par défaut vers le
pare-feu et les routes de retour vers les réseaux internes.
Configuration complète : [routeur.txt](./configs/routeur.txt)

## Fichier Packet Tracer

La topologie complète est téléchargeable : [infra-multivlan-dmz.pkt](./infra-multivlan-dmz.pkt)

Ouvrir avec Cisco Packet Tracer 8.x (gratuit avec un compte Cisco NetAcad).

## Tests de recette

| # | Test | Résultat attendu | Statut |
|---|------|------------------|--------|
| 1 | Poste VLAN 10 obtient une IP par DHCP | Bail à partir de 10.0.10.20 | OK |
| 2 | Ping entre deux postes du même VLAN | Réponse | OK |
| 3 | Ping inter-VLAN 10 vers 20 | Réponse via le switch L3 | OK |
| 4 | Ping poste interne vers 8.8.8.8 | Réponse via NAT | OK |
| 5 | Accès au serveur web en DMZ depuis le réseau interne | Page servie | OK |
| 6 | `show vlan brief` sur le switch L3 | VLAN 10/20/30/40/50 actifs | OK |

Captures dans [/captures](./captures/)

## Problèmes rencontrés

**Les pings sortants ne revenaient pas**
Symptôme : depuis un poste interne, aucune réponse à un ping vers Internet, alors que le
NAT était en place et que le pare-feu joignait lui-même 8.8.8.8.
Cause : l'ASA bloque par défaut le trafic de retour ICMP, qui n'est pas inspecté.
Correction : `inspect icmp` dans la policy-map globale.
Ce que j'en retire : sur un pare-feu à états, le sens d'ouverture d'un flux compte autant
que la règle elle-même.

**À compléter avec les autres incidents rencontrés**
Format : symptôme observé → ce que j'ai vérifié → cause → correction.

## Ce que je ferais différemment aujourd'hui

- **Aucune redondance** : la panne du switch L3 ou du pare-feu coupe tout. Un second lien
  avec HSRP et du spanning-tree correctement dimensionné changerait ça.
- **VLAN natif laissé en 1** sur les trunks, ce qui expose au VLAN hopping. Je le
  basculerais sur un VLAN dédié et inutilisé.
- **Pas de VLAN d'administration séparé** : les équipements sont joignables depuis les
  réseaux métier. Un VLAN 99 réservé à l'administration, accessible depuis le seul poste
  technicien, serait plus sain.
- **Pas de supervision** : aucun moyen de savoir qu'un lien est tombé avant que les
  utilisateurs appellent.
- **Accès en telnet** plutôt qu'en SSH avec comptes nominatifs.

## Environnement

Cisco Packet Tracer : switches d'accès 2960, switch de niveau 3, routeurs 1841,
pare-feu ASA, serveur DHCP, serveur web, postes clients.
Trunking 802.1Q, NAT/PAT, routes statiques, policy-map ASA.

