# Lab 02 — Active Directory, DNS, DHCP et GPO (pfSense en passerelle)

> 🟡 **En cours** — domaine, comptes, groupes et client opérationnels. Partages et GPO en cours de mise en place.

## 1. Objectif

Monter un petit système d'information d'entreprise tel qu'un technicien support N1/N2 le gère au quotidien :

- un domaine Active Directory avec une arborescence d'OU propre ;
- des comptes utilisateurs par service, des comptes d'administration séparés et un compte de service ;
- des groupes de sécurité organisés selon la méthode **AGDLP** ;
- DNS et DHCP portés par le contrôleur de domaine ;
- un poste client Windows 11 joint au domaine ;
- des partages par service et des GPO (à venir) ;
- des scénarios support réalistes : première connexion, mot de passe oublié, compte verrouillé, ajout à un groupe.

## 2. Environnement

| Élément | Détail |
|---|---|
| Hyperviseur | VMware Workstation |
| Pare-feu / passerelle | pfSense 2.9.0 — 2 Go RAM, 1 vCPU, 20 Go |
| Contrôleur de domaine | Windows Server 2022 Standard Évaluation (build 20348) |
| Poste client | Windows 11 Pro 25H2 |
| Domaine | `lab.local` (NetBIOS : `LAB`) |

## 3. Schéma et plan d'adressage

```
                    Internet
                       │
   ┌───────────────────┴───────────────────┐
   │  NAT VMware — VMnet8 192.168.100.0/24 │
   │  passerelle 192.168.100.2             │
   └───────────────────┬───────────────────┘
                       │
              em0 (WAN, DHCP 192.168.100.128)
              ┌────────┴────────┐
              │     pfSense     │
              └────────┬────────┘
              em1 (LAN 10.0.10.254/24)
                       │
   ┌───────────────────┴───────────────────┐
   │      VMnet2 — LAN 10.0.10.0/24        │
   └───────┬───────────────────────┬───────┘
           │                       │
     SRV-DC01                    PC-01
     10.0.10.10                  DHCP (10.0.10.100)
     AD DS · DNS · DHCP          Windows 11 Pro
```

### Réseaux VMware

| VMnet | Type | Réseau | Rôle |
|---|---|---|---|
| VMnet8 | NAT | 192.168.100.0/24, passerelle .2, DHCP .128 à .253 | WAN pfSense |
| VMnet2 | Personnalisé, sans DHCP, sans adaptateur hôte | 10.0.10.0/24 | LAN |

### Adressage

| Machine | IP | Passerelle | DNS |
|---|---|---|---|
| pfSense WAN (em0) | DHCP — 192.168.100.128 | - | - |
| pfSense LAN (em1) | 10.0.10.254/24 | - | - |
| SRV-DC01 | 10.0.10.10/24 (fixe) | 10.0.10.254 | lui-même, redirecteur vers 10.0.10.254 |
| PC-01 | DHCP (10.0.10.100) | 10.0.10.254 | 10.0.10.10 |

## 4. Mise en place

### 4.1 pfSense

Depuis la console pfSense :

- **Option 1 — Assign Interfaces** : pas de VLAN, WAN = `em0`, LAN = `em1`.
- **Option 2 — Set interface IP address** : LAN en `10.0.10.254/24`, sans passerelle amont, **sans serveur DHCP** (c'est le DC qui distribue les adresses).
- **Option 7 — Ping host** : vérification avec `8.8.8.8`.

### 4.2 Contrôleur de domaine — réseau

```powershell
Get-NetAdapter
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 10.0.10.10 -PrefixLength 24 -DefaultGateway 10.0.10.254
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 10.0.10.254
Rename-Computer -NewName SRV-DC01 -Restart
```

Tests : `ping 10.0.10.254`, `ping 8.8.8.8`, `ping google.fr`.

### 4.3 AD DS et DNS

Équivalent PowerShell de la promotion :

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "lab.local" -DomainNetbiosName "LAB" -InstallDns
```

Après redémarrage, redirecteur DNS vers pfSense :

```powershell
Set-DnsServerForwarder -IPAddress 10.0.10.254
```

### 4.4 DHCP

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
Add-DhcpServerInDC -DnsName "SRV-DC01.lab.local" -IPAddress 10.0.10.10
Add-DhcpServerv4Scope -Name "LAN" -StartRange 10.0.10.100 -EndRange 10.0.10.199 -SubnetMask 255.255.255.0
Set-DhcpServerv4OptionValue -ScopeId 10.0.10.0 -Router 10.0.10.254 -DnsServer 10.0.10.10 -DnsDomain "lab.local"
```

### 4.5 Arborescence d'OU

```
lab.local
└── LAB
    ├── Comptes_Admin          comptes d'administration nominatifs (adm.*)
    ├── Comptes_De_Services    svc_glpi
    ├── Désactivés             comptes des salariés partis
    ├── Groupes
    │   ├── Domaine_Local      DL_*
    │   └── Globaux            GG_*
    ├── Postes
    │   ├── Bureautique        PC-01
    │   └── Production
    ├── Serveurs
    │   ├── Applications
    │   ├── Fichiers
    │   └── Impression
    └── Utilisateurs
        ├── Compta  ├── Direction  ├── Externes  ├── Informatique  └── Production
```

Le contrôleur de domaine reste dans l'OU par défaut **Domain Controllers** (voir § 6).

### 4.6 Comptes

Tous les comptes sont créés **par script PowerShell** (`scripts/`) :

| Type | Détail |
|---|---|
| Utilisateurs | 18 comptes répartis en 5 services, changement de mot de passe obligatoire à la première connexion |
| Administration | 2 comptes séparés `adm.prenom.nom` pour les techniciens, distincts de leur compte de travail quotidien |
| Service | `svc_glpi` (lecture LDAP pour le futur lab GLPI) — mot de passe non expirant et non modifiable par l'utilisateur |
| Désactivé | 1 compte de salarié parti, déplacé dans `Désactivés` avec la date de départ en description |

### 4.7 Groupes — méthode AGDLP

Les **comptes** sont dans des **groupes globaux** (GG, par service), eux-mêmes membres de **groupes de domaine local** (DL, par ressource et niveau de droit). Les permissions ne sont jamais données directement aux utilisateurs.

| Dossier | Lecture (`DL_*_L`) | Modification (`DL_*_M`) | Contrôle total (`DL_*_CT`) |
|---|---|---|---|
| Commun | GG_Externes | GG_Compta, GG_Direction, GG_Informatique, GG_Production | GG_Admins_Serveurs |
| Compta | GG_Direction | GG_Compta | GG_Admins_Serveurs |
| Direction | — | GG_Direction | GG_Admins_Serveurs |
| Informatique | — | GG_Informatique | GG_Admins_Serveurs |
| Production | GG_Direction | GG_Production | GG_Admins_Serveurs |

Autres groupes :

- `DL_Helpdesk_ResetMdp` ← `GG_Informatique` : délégation de la réinitialisation des mots de passe ;
- `DL_Serveurs_AdminLocal` ← `GG_Admins_Serveurs` : administrateurs locaux des serveurs.

Vérification :

```powershell
Get-ADGroup -Filter * -SearchBase "OU=Domaine_Local,OU=Groupes,OU=LAB,DC=lab,DC=local" |
  ForEach-Object { [PSCustomObject]@{ Groupe = $_.Name; Membres = (Get-ADGroupMember $_ | Select-Object -ExpandProperty Name) -join ", " } } |
  Sort-Object Groupe | Format-Table -AutoSize
```

### 4.8 Poste client

- PC-01 en DHCP : reçoit 10.0.10.100, passerelle .254, DNS .10, suffixe `lab.local`.
- Test du port LDAP avant jonction : `Test-NetConnection 10.0.10.10 -Port 389`.
- Jonction au domaine avec un **compte d'administration nominatif** (`LAB\adm.marc.fontaine`), pas avec le compte Administrateur.
- Objet ordinateur rangé dans `Postes\Bureautique`.

## 5. Tests de recette

| Test | Commande / action | Résultat |
|---|---|---|
| Internet depuis pfSense | Console, option 7 → `8.8.8.8` | ✅ 0 % de perte |
| Bail DHCP sur le client | `ipconfig /all` | ✅ 10.0.10.100, DNS 10.0.10.10 |
| Joignabilité LDAP | `Test-NetConnection 10.0.10.10 -Port 389` | ✅ |
| Jonction au domaine | Propriétés système | ✅ « Bienvenue dans le domaine lab.local » |
| Ouverture de session domaine | `LAB\emma.lambert` sur PC-01 | ✅ authentifiée par `\\SRV-DC01` |
| Changement de mot de passe à la 1re connexion | Ouverture de session | ✅ |
| Jeton de groupes AGDLP | `whoami /groups \| findstr "GG_ DL_"` | ✅ GG_Informatique → DL_Commun_M, DL_Informatique_M, DL_Helpdesk_ResetMdp |
| GPO appliquées sur le DC | `gpresult /r /scope computer` | ✅ Default Domain Controllers Policy + Default Domain Policy |
| Accès partage autorisé / refusé | `dir \\SRV-DC01\...` | ⏳ à venir |
| GPO lecteurs réseau, panneau de configuration, mots de passe | `gpresult /r` sur PC-01 | ⏳ à venir |
| Scénarios support (reset mot de passe, déverrouillage, ajout à un groupe) | — | ⏳ à venir |

### Captures

![Console pfSense WAN + LAN](captures/02-pfsense-console-wan-lan.png)
![Arborescence AD](captures/03-ad-arborescence-ou.png)
![Groupes AGDLP](captures/04-agdlp-groupes-membres.png)
![Options DHCP](captures/05-dhcp-options-etendue.png)
![Jonction au domaine](captures/06-pc01-jonction-domaine.png)
![Groupes de l'utilisateur](captures/08-whoami-groups-emma.png)

## 6. Problèmes rencontrés

| Symptôme | Cause | Solution |
|---|---|---|
| Impossible d'utiliser 10.0.10.0/24 pour le LAN | Le NAT VMware (VMnet8) occupait déjà ce sous-réseau | VMnet8 déplacé en 192.168.100.0/24 |
| L'installeur pfSense ne trouvait pas Internet | Interfaces WAN/LAN inversées (em0/em1) | Diagnostic avec une seule carte réseau, puis réaffectation |
| Toujours pas d'Internet avec une seule carte | Passerelle NAT VMware en .254, à l'intérieur de la plage DHCP (.128 à .254) | Passerelle en .2, plage DHCP .128 à .253, redémarrage des services VMware |
| Risque de plantage de l'installeur | VM créée avec 256 Mo de RAM | Passée à 2 Go |
| LAN absent au démarrage de pfSense (seul le WAN affiché, em1 « down ») | Installation faite avec une seule carte (phase de diagnostic), LAN jamais assigné | Console option 1 (WAN em0 / LAN em1), puis option 2 : LAN 10.0.10.254/24 sans DHCP |
| Contrôleur de domaine rangé dans `Serveurs\Fichiers` | Objet déplacé hors de l'OU Domain Controllers → plus de *Default Domain Controllers Policy* | `Get-ADComputer SRV-DC01 \| Move-ADObject -TargetPath "OU=Domain Controllers,DC=lab,DC=local"` puis `gpupdate /force` |
| « Impossible de supprimer SRV-DC01 : objet non trouvé » dans la console AD | Console non rafraîchie après le `Move-ADObject` : l'objet affiché n'existait plus à cet endroit | F5 dans la console. Un DC mal rangé se **déplace**, il ne se supprime jamais |

## 7. Ce que j'ai appris

- **Isoler les problèmes réseau un par un** : retirer une carte, tester, puis remettre. C'est plus rapide que de tout modifier d'un coup.
- **Une plage DHCP ne doit jamais contenir l'adresse de la passerelle.**
- **AGDLP** : on ne donne jamais de droits à un utilisateur. On ajoute la personne à son groupe de service, et l'accès suit. Un départ ou une arrivée se gère en une seule action.
- **Séparer compte de travail et compte d'administration** limite les dégâts en cas de compromission d'une session utilisateur.
- **Le DC reste dans l'OU Domain Controllers** : la déplacer lui fait perdre sa stratégie de sécurité.
- **Rafraîchir la console AD (F5)** après une modification faite en PowerShell, avant toute autre action.
- Dans un lab, les partages sont sur le DC. En production, on les mettrait sur un serveur de fichiers dédié.

## Prochaines étapes

- [ ] Assistant web pfSense (hostname, domaine, fuseau, mot de passe admin)
- [ ] Partages par service avec droits NTFS AGDLP et énumération basée sur l'accès
- [ ] 3 GPO : lecteur réseau par service, blocage du panneau de configuration, politique de mots de passe
- [ ] Délégation helpdesk : réinitialisation et déverrouillage des comptes par le groupe Informatique, sans droits admin
- [ ] Scénarios support documentés pas à pas
- [ ] Export des GPO (`Backup-GPO -All -Path C:\GPO`)
