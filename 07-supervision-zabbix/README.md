# Lab 07 — Supervision d'une infrastructure avec Zabbix

> **Statut : 🟡 En cours** — supervision par ping, carte et tableau de bord opérationnels. Prochaine étape : Zabbix Agent 2 sur les postes Windows et SNMP sur pfSense.

![Tableau de bord Homelab](captures/05-tableau-de-bord.png)

## 1. Objectif

Savoir **qu'une machine est tombée avant que l'utilisateur appelle le support**, et visualiser l'infrastructure d'un coup d'œil.

Au quotidien, un technicien support utilise un outil de supervision pour :

- détecter une panne (poste, serveur, pare-feu injoignable) ;
- voir **où** se trouve la machine concernée (carte du site) ;
- prendre en charge l'incident (acquittement, commentaire) et suivre sa résolution ;
- diagnostiquer à distance (ping, traceroute) sans se déplacer.

## 2. Environnement

| Élément | Détail |
|---|---|
| Hyperviseur | VMware Workstation |
| Pare-feu / passerelle | pfSense 2.9.0 (`fw01`) |
| Serveur | Windows Server 2022 (`SRV-DC01`) |
| Poste client | Windows 11 Pro (`PC-01`) |
| Supervision | Debian 13, Zabbix **7.0.32 LTS** installé par paquets, 4 Go RAM, 2 vCPU |
| Réseau | LAN isolé VMnet2 `10.0.10.0/24`, accès Internet via pfSense |

## 3. Schéma et plan d'adressage

La carte Zabbix est posée sur un **plan d'usine fictif** (« ACME Industrie ») pour situer chaque équipement dans un site réaliste. Aucun plan réel n'est utilisé.

![Carte Zabbix — tout est OK](captures/02-carte-ok.png)

| Machine | Rôle | Emplacement (carte) | IP |
|---|---|---|---|
| `fw01` | Pare-feu pfSense, passerelle | Salle serveur | 10.0.10.254 |
| `SRV-DC01` | Contrôleur de domaine | Salle serveur | 10.0.10.10 |
| Zabbix | Serveur de supervision | Salle serveur | 10.0.10.101 |
| `PC-01` | Poste utilisateur | Bureaux | 10.0.10.100 |

L'**atelier production** de la carte est volontairement vide : il accueillera l'automate du lab 08 (segmentation IT/OT).

## 4. Mise en place

### 4.1 Autoriser le ping sur les machines Windows

Par défaut, le pare-feu Windows bloque les requêtes ICMP entrantes. Sur `SRV-DC01` et `PC-01`, en PowerShell administrateur :

```powershell
New-NetFirewallRule -DisplayName "ICMPv4 Echo In" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow
```

Test **depuis une autre machine** (le serveur Zabbix) :

```bash
ping -c 2 10.0.10.254 && ping -c 2 10.0.10.10 && ping -c 2 10.0.10.100
```

### 4.2 Déclarer les hôtes dans Zabbix

*Collecte de données → Hôtes → Créer un hôte*

| Nom de l'hôte | Groupe | Interface | Modèle |
|---|---|---|---|
| `fw01` | Homelab | 10.0.10.254 | ICMP Ping |
| `SRV-DC01` | Homelab | 10.0.10.10 | ICMP Ping |
| `PC-01` | Homelab | 10.0.10.100 | ICMP Ping |

Le modèle **ICMP Ping** remonte 3 indicateurs : disponibilité (Up/Down), perte de paquets, temps de réponse.

![Dernières données — les 3 hôtes répondent](captures/01-dernieres-donnees.png)

### 4.3 Créer la carte

1. *Administration → Général → Images* → type **Fond** → import du plan fictif (1200×700 px).
2. *Surveillance → Cartes → Créer une carte* : 1200×700, image d'arrière-plan = le plan.
3. Ajout des éléments (type Hôte, icônes Router / Server / Workstation), étiquettes `{HOST.NAME}` et `{HOST.IP}`.
4. Liens Internet → fw01 → chaque machine.
5. **Indicateurs de lien** : sur chaque lien, déclencheur `Unavailable by ICMP ping` de la machine au bout du lien, couleur rouge. Le lien passe au rouge uniquement si **cette** machine tombe.

### 4.4 Activer les scripts de diagnostic (Ping / Traceroute depuis la carte)

Zabbix 7.0 désactive par défaut l'exécution des scripts globaux. Sur le serveur Zabbix :

```bash
sed -i '/EnableGlobalScripts/d' /etc/zabbix/zabbix_server.conf
echo "EnableGlobalScripts=1" >> /etc/zabbix/zabbix_server.conf
systemctl restart zabbix-server
```

Un clic sur une machine de la carte → **Ping** lance le test depuis le serveur de supervision.

![Ping lancé depuis la carte](captures/06-script-ping.png)

### 4.5 Tableau de bord

Tableau de bord **Homelab** : carte du site + widget **Problèmes** (groupes Homelab et Zabbix servers) + graphique du temps de réponse ICMP.

## 5. Tests de recette

| Test | Résultat attendu | Résultat |
|---|---|---|
| Ping des 3 machines depuis Zabbix | 0 % de perte | ✅ |
| Hôtes visibles dans *Dernières données* | ICMP ping = Up (1) | ✅ |
| Arrêt de la VM `PC-01` | Icône + lien fw01 ↔ PC-01 en rouge, problème « Unavailable by ICMP ping » en 1 à 3 min | ✅ |
| Les autres liens pendant la panne | Restent verts | ✅ |
| Redémarrage de `PC-01` | Retour au vert, problème passé en **RÉSOLU** | ✅ |
| Ping depuis la carte | Résultat affiché dans la fenêtre | ✅ |

**Panne simulée sur PC-01 :**

![Carte — PC-01 en panne](captures/03-carte-panne-pc01.png)

**Historique des incidents (détection → prise en charge → résolution) :**

![Problèmes résolus](captures/04-problemes.png)

## 6. Problèmes rencontrés

| Symptôme | Cause | Solution |
|---|---|---|
| Zabbix ne pingue pas `PC-01` | Le pare-feu Windows bloque l'ICMP entrant par défaut | Règle `New-NetFirewallRule` sur chaque machine Windows ; test depuis une **autre** machine (un ping vers soi-même ne prouve rien) |
| Fond de carte coupé | Carte créée en 800×600 alors que l'image fait 1200×700 | Taille de la carte alignée sur celle de l'image |
| « L'exécution du script global sur le serveur Zabbix est désactivée » | Zabbix 7.0 désactive les scripts globaux par défaut (ils s'exécutent sur le serveur) | `EnableGlobalScripts=1` + redémarrage du service ; en production, limiter les scripts et les groupes autorisés |
| Panne de `PC-01` : l'icône passe en rouge mais le lien reste vert | Aucun indicateur de lien (un lien ne suit pas automatiquement l'état des hôtes) | Indicateur `Unavailable by ICMP ping` ajouté sur le lien |
| Panne de `PC-01` : **tous** les liens passent au rouge | Le déclencheur de PC-01 avait été ajouté sur tous les liens | Vérification lien par lien : un seul déclencheur, celui de la machine au bout du lien |
| Alerte « High swap space usage (less than 50% free) » sur le serveur Zabbix | VM à 2 Go de RAM, saturée par le navigateur et l'interface graphique lancés dans la VM (diagnostic : `free -h`, `ps aux --sort=-%mem`) | RAM passée à 4 Go ; à terme, consultation de Zabbix depuis le poste client et serveur sans interface graphique |

## 7. Ce que j'ai appris

- Une supervision ne vaut que si elle est **testée** : couper volontairement une machine et vérifier que l'alerte remonte, puis qu'elle se résout.
- Les sécurités par défaut (pare-feu Windows, scripts désactivés dans Zabbix) sont normales : on les ouvre **en sachant pourquoi**, et au minimum nécessaire.
- Un outil de supervision peut lui-même lever des alertes sur sa propre machine (ici, le swap) : il faut savoir lire `free -h` et identifier ce qui consomme.
- Une carte bien construite (indicateurs de lien précis, emplacements réalistes) permet à un technicien de localiser une panne en quelques secondes.
- Le cycle d'un incident dans l'outil : détection → acquittement avec commentaire → résolution → historique.

## 8. Prochaines étapes

- [ ] Zabbix Agent 2 sur `SRV-DC01` et `PC-01` (CPU, RAM, disques, services AD/DNS)
- [ ] SNMP sur pfSense (interfaces, trafic)
- [ ] Fuseau horaire Europe/Paris (serveur + interface Zabbix)
- [ ] Notifications par e-mail sur les problèmes de sévérité Haut
- [ ] Serveur de supervision sans interface graphique, consulté depuis le poste client
