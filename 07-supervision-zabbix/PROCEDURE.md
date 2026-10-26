# Procédure — Installer Zabbix 7.0 et superviser une petite infrastructure

> Lab 07 du homelab Stev-J. Procédure reproductible, pas à pas, avec les erreurs rencontrées et leur solution.

## Sommaire

1. [Prérequis](#1-prérequis)
2. [Installer Zabbix sur Debian 13](#2-installer-zabbix-sur-debian-13)
3. [Préparer la base PostgreSQL](#3-préparer-la-base-postgresql)
4. [Configurer et démarrer les services](#4-configurer-et-démarrer-les-services)
5. [Assistant d'installation web](#5-assistant-dinstallation-web)
6. [Sécuriser les accès par défaut](#6-sécuriser-les-accès-par-défaut)
7. [Autoriser le ping sur les machines Windows](#7-autoriser-le-ping-sur-les-machines-windows)
8. [Déclarer les hôtes (ICMP)](#8-déclarer-les-hôtes-icmp)
9. [Superviser pfSense en SNMP](#9-superviser-pfsense-en-snmp)
10. [Créer la carte sur un plan d'usine](#10-créer-la-carte-sur-un-plan-dusine)
11. [Activer Ping / Traceroute depuis la carte](#11-activer-ping--traceroute-depuis-la-carte)
12. [Tableau de bord](#12-tableau-de-bord)
13. [Test de recette](#13-test-de-recette)
14. [Problèmes rencontrés](#14-problèmes-rencontrés)

---

## 1. Prérequis

| Élément | Valeur |
|---|---|
| VM supervision | Debian 13, **4 Go RAM** (2 Go ne suffisent pas avec l'interface graphique), 2 vCPU, 20 Go, réseau VMnet2 |
| IP | 10.0.10.101/24, passerelle 10.0.10.254 |
| Machines supervisées | `fw01` pfSense 10.0.10.254, `SRV-DC01` 10.0.10.10, `PC-01` 10.0.10.100 |
| Accès | compte root (`su -`) |

Vérifier la version de Debian et l'IP :

```bash
cat /etc/debian_version
ip a
```

> ⚠️ Zabbix fournit des paquets différents selon la distribution. Les commandes `rpm` (Rocky/RHEL) ne fonctionnent pas sur Debian : il faut choisir **Debian 13 / PostgreSQL / Nginx** sur la page de téléchargement de Zabbix.

## 2. Installer Zabbix sur Debian 13

Ajouter le dépôt Zabbix 7.0 (version LTS) :

```bash
wget https://repo.zabbix.com/zabbix/7.0/debian/pool/main/z/zabbix-release/zabbix-release_latest+debian13_all.deb
dpkg -i zabbix-release_latest+debian13_all.deb
apt update
```

<img width="1295" height="1003" alt="Dépôt Zabbix" src="https://github.com/user-attachments/assets/986458a8-e88d-4266-b406-822b2ef02de6" />

Installer le serveur, l'interface web, PostgreSQL, l'agent et les outils SNMP :

```bash
apt install -y zabbix-server-pgsql zabbix-frontend-php php8.4-pgsql zabbix-nginx-conf \
  zabbix-sql-scripts zabbix-agent2 postgresql snmp
```

> `php8.4-pgsql` n'est **pas** installé automatiquement : sans lui, l'assistant web ne propose que MySQL (voir §14).
> `snmp-mibs-downloader` n'existe pas sur Debian 13 : on s'en passe (voir §9).

<img width="2324" height="1286" alt="install-paquets" src="https://github.com/user-attachments/assets/d31bb68a-893c-43d7-ad79-be0d4946a992" />

## 3. Préparer la base PostgreSQL

Créer l'utilisateur et la base (un mot de passe est demandé, à noter pour le §4) :

```bash
su - postgres -c "createuser --pwprompt zabbix"
su - postgres -c "createdb -O zabbix zabbix"
```

Importer le schéma **en tant qu'utilisateur système `zabbix`** (les tables appartiennent directement au bon propriétaire) :

```bash
zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql zabbix
```

L'import dure 1 à 2 minutes et défile beaucoup de `CREATE TABLE` / `INSERT`.

<img width="2314" height="1311" alt="Base-postgresql" src="https://github.com/user-attachments/assets/0f4366b7-c73b-4c60-a496-cfaebafcb75e" />

## 4. Configurer et démarrer les services

Renseigner le mot de passe de la base :

```bash
nano /etc/zabbix/zabbix_server.conf
```

```ini
DBPassword=<mot de passe choisi au §3>
```

Configurer Nginx (décommenter les deux lignes) :

```bash
nano /etc/zabbix/nginx.conf
```

```nginx
listen 8080;
server_name _;
```

Démarrer et activer au démarrage :

```bash
systemctl restart zabbix-server zabbix-agent2 nginx php8.4-fpm
systemctl enable zabbix-server zabbix-agent2 nginx php8.4-fpm
systemctl status zabbix-server
```

Le service doit être active (running). En cas d'échec, lire le journal :

```bash
tail -n 30 /var/log/zabbix/zabbix_server.log
```

<img width="2329" height="1262" alt="Service-actif" src="https://github.com/user-attachments/assets/93a2e532-8ba1-447c-ae88-8078088251ae" />

## 5. Assistant d'installation web

Depuis un navigateur : `http://10.0.10.101:8080`

| Étape | Réglage |
|---|---|
| Bienvenue | Langue : Français |
| Prérequis | Tout doit être **OK** |
| Base de données | Type **PostgreSQL**, hôte `localhost`, port `0`, base `zabbix`, utilisateur `zabbix`, mot de passe du §3 |
| Paramètres | Nom du serveur : `Homelab` — Fuseau horaire : **Europe/Paris** (sinon les heures sont décalées d'1 h) |
| Résumé | Vérifier, puis *Prochaine étape* |

<img width="1302" height="1055" alt="Setup-prerequis" src="https://github.com/user-attachments/assets/725f03fa-2520-4dd9-babd-20f7f0718ac0" />
<img width="1004" height="678" alt="setup-erreur-mysql" src="https://github.com/user-attachments/assets/0b7eb9f8-c0a8-4757-aea8-33362788ea21" />
<img width="972" height="617" alt="setup-postgresql" src="https://github.com/user-attachments/assets/e0508133-a76a-46de-9704-c2fe95274a1b" />
<img width="896" height="480" alt="setup-resume" src="https://github.com/user-attachments/assets/2d099993-8856-415c-8139-384732f2518a" />

Connexion avec les identifiants par défaut de l'interface : **Admin / zabbix** (ce ne sont pas ceux de la base de données).

<img width="2548" height="1364" alt="premier-tableau-de-bord" src="https://github.com/user-attachments/assets/18213e6c-69e6-41d6-bf48-3563dee3450c" />

## 6. Sécuriser les accès par défaut

À faire avant d'aller plus loin :

| Quoi | Où | Action |
|---|---|---|
| Compte Zabbix `Admin` | *Utilisateurs → Utilisateurs → Admin → Changer le mot de passe* | Mot de passe fort |
| Compte pfSense `admin` | pfSense : *System → User Manager* | Remplacer le mot de passe par défaut `pfsense` |
| Fuseau horaire Debian | terminal | `timedatectl set-timezone Europe/Paris` |
| Fuseau horaire Zabbix | *Administration → Général → Interface graphique* | Europe/Paris |

## 7. Autoriser le ping sur les machines Windows

Le pare-feu Windows bloque l'ICMP entrant par défaut. Sur **SRV-DC01** et **PC-01**, en PowerShell administrateur :

```powershell
New-NetFirewallRule -DisplayName "ICMPv4 Echo In" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow
```

Tester **depuis une autre machine** (un ping vers soi-même répond toujours) :

```powershell
ping -n 2 10.0.10.100     # depuis SRV-DC01 (Windows : -n)
```

```bash
ping -c 2 10.0.10.100     # depuis la Debian (Linux : -c)
```

<img width="970" height="518" alt="regle-icmp" src="https://github.com/user-attachments/assets/81073e83-6142-4dcb-aa98-d3a787e0e122" />
<img width="828" height="583" alt="ping-debian" src="https://github.com/user-attachments/assets/576d2a5c-4148-476c-815f-d0cdfdc122b3" />

## 8. Déclarer les hôtes (ICMP)

*Collecte de données → Hôtes → Créer un hôte*

| Nom de l'hôte | Modèle | Groupe | Interface |
|---|---|---|---|
| `fw01` | ICMP Ping | Homelab | Agent — 10.0.10.254 |
| `SRV-DC01` | ICMP Ping | Homelab | Agent — 10.0.10.10 |
| `PC-01` | ICMP Ping | Homelab | Agent — 10.0.10.100 |

Le modèle ICMP Ping utilise `fping` (installé au §2) et remonte 3 valeurs : disponibilité, perte, temps de réponse.

<img width="1271" height="770" alt="hote-icmp" src="https://github.com/user-attachments/assets/81e9444c-f15e-4d0b-a345-49639329f6c8" />

Vérifier après 1 à 2 minutes : *Surveillance → Dernières données*, filtre groupe **Homelab**.

<img width="2108" height="1166" alt="dernieres-donnees" src="https://github.com/user-attachments/assets/0b73a298-e35a-4702-a2a4-08d7d3c91acb" />

## 9. Superviser pfSense en SNMP

**Sur pfSense** (*Services → SNMP*) :

| Champ | Valeur |
|---|---|
| Enable | ☑ |
| Polling Port | 161 |
| Read Community String | une chaîne personnelle (pas `public`) |
| Bind Interfaces | **LAN** uniquement (pas All, pour ne pas exposer SNMP côté WAN) |

<img width="1160" height="1113" alt="snmp-pfsense" src="https://github.com/user-attachments/assets/37bd56e7-c099-4344-8ddc-2646ff481797" />

**Test depuis la Debian** (OID numérique, car les MIB ne sont pas installées) :

```bash
snmpwalk -v2c -c <communauté> 10.0.10.254 1.3.6.1.2.1.1
```

Réponse attendue : `STRING: "pfSense ... 2.9.0-RELEASE ..."`.

<img width="1457" height="1115" alt="snmpwalk" src="https://github.com/user-attachments/assets/84d14114-d6de-44ad-b3b9-a92775bb663c" />

**Dans Zabbix**, sur l'hôte `fw01` :

1. Ajouter le modèle **Network Generic Device by SNMP**.
2. Interfaces → *Ajouter* → **SNMP** : IP 10.0.10.254, port 161, SNMPv2, communauté `{$SNMP_COMMUNITY}`.
3. Onglet *Macros* : `{$SNMP_COMMUNITY}` = la chaîne choisie sur pfSense.

Le modèle découvre automatiquement les interfaces (**em0 = WAN**, **em1 = LAN**) : trafic, erreurs, statut, débit, uptime.

<img width="2216" height="1215" alt="elements-snmp" src="https://github.com/user-attachments/assets/2a487fce-6930-4f20-980a-9b5fcf02037b" />

## 10. Créer la carte sur un plan d'usine

**Importer le fond** : *Administration → Général → Images* → type **Fond** (pas Icône) → *Créer un fond*.
Le plan `usine-fictive.png` (1200×700, plan fictif) est fourni dans ce dossier.

<img width="1759" height="1033" alt="fond-importe" src="https://github.com/user-attachments/assets/ac6f2b85-1e8d-4cb3-801c-63d3303da94d" />

**Créer la carte** : *Surveillance → Cartes → Créer une carte* — Largeur **1200**, Hauteur **700** (mêmes dimensions que l'image), image de fond = le plan.

**Placer les éléments** (*Éditer la carte → Élément de carte : Ajouter*, clic sur l'élément) :

| Type | Hôte | Icône | Étiquette | X | Y |
|---|---|---|---|---|---|
| Image | — | Cloud_(64) | `Internet` | 450 | 10 |
| Hôte | fw01 | Router_(64) | `{HOST.NAME}` ↵ `{HOST.IP}` | 440 | 130 |
| Hôte | SRV-DC01 | Server_(64) | `{HOST.NAME}` ↵ `{HOST.IP}` | 550 | 250 |
| Hôte | Zabbix server | Server_(64) | `Zabbix` ↵ `10.0.10.101` | 400 | 250 |
| Hôte | PC-01 | Workstation_(64) | `{HOST.NAME}` ↵ `{HOST.IP}` | 150 | 250 |

*Substituer les macros : Actif* pour voir les vrais noms pendant l'édition.

<img width="1043" height="807" alt="element-carte" src="https://github.com/user-attachments/assets/49aeb835-f897-42a9-a23c-058448d6d6e1" />

**Tracer les liens** : clic sur un élément, **Ctrl + clic** sur le second, *Lier : Ajouter*.
Internet ↔ fw01, fw01 ↔ SRV-DC01, fw01 ↔ Zabbix, fw01 ↔ PC-01.

**Indicateurs de lien** (le lien passe au rouge si la machine tombe) : sélectionner la paire → *Liens : Édition* → *Indicateurs de lien : Ajouter* → hôte **de la machine au bout du lien** → `Unavailable by ICMP ping`, couleur `DD0000`.
Un seul déclencheur par lien, celui de sa machine.

<img width="936" height="943" alt="203" src="https://github.com/user-attachments/assets/e7ff3d45-ed12-439c-8598-4ad4c3a694f8" />
<img width="890" height="568" alt="204" src="https://github.com/user-attachments/assets/92dbdf57-49be-47e5-a8dc-6ef84adba116" />
<img width="882" height="270" alt="205" src="https://github.com/user-attachments/assets/662eaad8-5b75-4e03-b969-44b0a2e3b8a4" />

Cliquer sur **Actualiser** pour enregistrer la carte.

<img width="1449" height="856" alt="carte-ok" src="https://github.com/user-attachments/assets/348c1ea1-b9d3-4455-a395-da6147b6795d" />

## 11. Activer Ping / Traceroute depuis la carte

Zabbix 7.0 désactive par défaut les scripts globaux (ils s'exécutent sur le serveur). Pour un lab :

```bash
sed -i '/EnableGlobalScripts/d' /etc/zabbix/zabbix_server.conf
echo "EnableGlobalScripts=1" >> /etc/zabbix/zabbix_server.conf
systemctl restart zabbix-server
```

Clic sur une machine de la carte → **Ping** : le résultat s'affiche.

<img width="1764" height="544" alt="script-ping" src="https://github.com/user-attachments/assets/eece39f3-1a99-4c99-a2e5-aadb5f1ee0c7" />

## 12. Tableau de bord

*Tableaux de bord → Créer un tableau de bord* → `Homelab`, puis ajouter :

| Widget | Réglages |
|---|---|
| Carte | Topologie Homelab (agrandir le widget pour voir tout le plan) |
| Problèmes | Nom `Incidents Homelab`, groupes **Homelab** + **Zabbix servers**, Problèmes récents |
| Graphique | Hôtes fw01, SRV-DC01, PC-01 — élément `ICMP response time` |

*Sauvegarder les modifications*.

<img width="2093" height="1227" alt="tableau-de-bord" src="https://github.com/user-attachments/assets/6df68f9f-a0ca-480d-af40-f1b51aaf9a29" />

## 13. Test de recette

1. Carte toute verte (capture §10).
2. Éteindre la VM **PC-01**.
3. Après 1 à 3 minutes : icône PC-01 + **seul** le lien fw01 ↔ PC-01 en rouge, problème « Unavailable by ICMP ping » (sévérité Haut).
4. Dans *Surveillance → Problèmes* : *Actualiser* sur la ligne → acquitter avec un message (ex. « VM éteinte pour test de recette »).
5. Rallumer PC-01 : tout repasse au vert, le problème passe en **RÉSOLU**.

<img width="1318" height="618" alt="carte-panne-pc01" src="https://github.com/user-attachments/assets/cb6a9da3-b579-4528-9319-40d0e38f9f2b" />

*VM PC-01 éteinte volontairement : seul le lien fw01 ↔ PC-01 passe au rouge, détection en moins de 3 minutes.*

<img width="1965" height="371" alt="217" src="https://github.com/user-attachments/assets/aadfbad9-7f7b-47e2-8966-afd87752bf10" />

*Historique des incidents : 2 pannes simulées de PC-01 (résolues au redémarrage) et une alerte mémoire sur le serveur Zabbix (résolue en passant la VM à 4 Go), prise en charge avec un commentaire.*

## 14. Problèmes rencontrés

| Symptôme | Cause | Solution |
|---|---|---|
| `rpm : commande introuvable` | Commandes du dépôt Rocky/RHEL utilisées sur Debian | Dépôt Debian 13 (`wget` + `dpkg -i`) |
| `snmp-mibs-downloader` introuvable | Paquet absent de Debian 13 | Installer `snmp` seul, interroger avec les OID numériques |
| Import du schéma : « authentification peer échouée pour l'utilisateur zabbix » | `psql -U zabbix` lancé depuis le compte système `postgres` : PostgreSQL exige que l'utilisateur système et l'utilisateur de la base correspondent | Importer avec `sudo -u zabbix psql zabbix` |
| zabbix-server ne démarre pas : « droit refusé pour la table users » / « not a Zabbix database » | Schéma importé sous `postgres` : tables inaccessibles à `zabbix` | `GRANT ALL` sur schéma, tables et séquences + `ALTER SCHEMA public OWNER TO zabbix`, puis redémarrage |
| Assistant web : seul MySQL proposé, « Impossible de se connecter à la base de données » | Module PHP PostgreSQL absent | `apt install php8.4-pgsql` + `systemctl restart php8.4-fpm` |
| Connexion refusée avec l'utilisateur `zabbix` | Confusion entre compte de la base et compte de l'interface | Interface : `Admin` / `zabbix` (à changer aussitôt) |
| Heures décalées d'1 h | Fuseau UTC choisi à l'installation | `timedatectl set-timezone Europe/Paris` + fuseau dans *Administration → Général* |
| `snmpwalk ... system` : « Unknown Object Identifier » | MIB non installées | OID numérique `1.3.6.1.2.1.1` |
| PC-01 ne répond pas au ping | Pare-feu Windows : ICMP entrant bloqué | Règle `New-NetFirewallRule` (§7) |
| `ping -c` refusé sous Windows | Syntaxe Linux | Windows : `ping -n 2` |
| Fond de carte coupé | Carte en 800×600, image en 1200×700 | Carte redimensionnée en 1200×700 |
| « L'exécution du script global sur le serveur Zabbix est désactivée » | Désactivé par défaut depuis Zabbix 7.0 | `EnableGlobalScripts=1` (§11) |
| Panne de PC-01 : icône rouge, lien vert | Pas d'indicateur de lien | Indicateur « Unavailable by ICMP ping » sur le lien |
| Panne de PC-01 : **tous** les liens rouges | Déclencheur de PC-01 ajouté sur tous les liens | Vérification lien par lien : un déclencheur par lien |
| Alerte « High swap space usage (less than 50% free) » | VM à 2 Go, saturée par Firefox + GNOME (`free -h`, `ps aux --sort=-%mem`) | RAM passée à 4 Go ; à terme, Zabbix consulté depuis PC-01 et Debian sans interface graphique |

---

### Captures à ne jamais publier

- Les fenêtres Firefox « Enregistrer le mot de passe » (mot de passe affiché en clair).
- Toute capture montrant un mot de passe, une communauté SNMP ou le fichier `zabbix_server.conf` avec `DBPassword`.
