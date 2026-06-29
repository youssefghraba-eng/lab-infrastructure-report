# 🛡️ SOC Lab Project — Sécurité Opérationnelle



 Youssef
**Date :** 28 juin 2026
**Type de projet :** Laboratoire de cybersécurité (SOC Lab) — Supervision, Audit et Durcissement système

---

## 📑 Table des matières

1. [Résumé du projet](#-résumé-du-projet)
2. [Architecture technique](#️-architecture-technique)
3. [Partie 1 — Préparation du centre de commande (Ubuntu Server)](#️-partie-1--préparation-du-centre-de-commande-ubuntu-server)
4. [Partie 2 — Migration vers Docker](#-partie-2--migration-vers-docker-solution-stable-et-professionnelle)
5. [Partie 3 — Préparation de la cible (Kali Linux)](#-partie-3--préparation-de-la-cible-kali-linux)
6. [Partie 4 — Liaison sécurisée SSH](#-partie-4--liaison-sécurisée-entre-ubuntu-server-et-kali-linux-ssh)
7. [Partie 5 — Audit de sécurité avec Lynis](#️-partie-5--audit-de-sécurité-avec-lynis)
8. [Partie 6 — Mesures correctives appliquées](#-partie-6--mesures-correctives-appliquées-remediation)
9. [Tableau récapitulatif](#-tableau-récapitulatif-des-problèmes-et-solutions)
10. [Conclusion](#-conclusion)

---

## 🚀 Résumé du projet

Ce dépôt documente, étape par étape, la construction d'un **laboratoire SOC (Security Operations Center)** complet :

- Un **centre de commande** (Ubuntu Server) hébergeant la stack **Wazuh** (SIEM open-source) sous **Docker**.
- Une **machine cible** (Kali Linux) supervisée via un agent Wazuh.
- Une **liaison SSH chiffrée par clé** entre les deux machines, sans authentification par mot de passe.
- Un **audit de sécurité** réalisé avec **Lynis**, suivi de la correction des vulnérabilités détectées.

L'objectif pédagogique et technique est de simuler un environnement réel de supervision de la sécurité (SIEM), incluant les difficultés rencontrées en conditions réelles (espace disque, conflits de versions, durcissement système) et la manière dont elles ont été résolues — chaque choix technique étant justifié.

---

## 🛠️ Architecture technique

| Composant | Détail |
|---|---|
| **Centre de commande** | Ubuntu Server 26.04 |
| **Cible supervisée** | Kali Linux (machine virtuelle) |
| **Hyperviseur** | Hyper-V |
| **Stack de supervision (SIEM)** | Wazuh Manager / Indexer / Dashboard — v4.14.0 |
| **Conteneurisation** | Docker Engine + Docker Compose Plugin |
| **Outil d'audit système** | Lynis |
| **Liaison sécurisée** | SSH (clé RSA 4096 bits) |
| **Adresse IP de la cible (Kali)** | `172.27.172.52` |

**Schéma logique :**

```
┌────────────────────────┐        SSH (clé RSA)        ┌────────────────────────┐
│   Ubuntu Server         │ ───────────────────────────▶│   Kali Linux            │
│   (Centre de commande)  │                              │   (Cible supervisée)    │
│                          │                              │                          │
│  ┌────────────────────┐ │        Agent Wazuh (logs)    │  ┌────────────────────┐ │
│  │ Wazuh Manager       │◀├───────────────────────────────┤  Wazuh Agent        │ │
│  │ Wazuh Indexer       │ │                              │  Lynis (audit)       │ │
│  │ Wazuh Dashboard     │ │                              │                      │ │
│  │ (via Docker Compose)│ │                              │                      │ │
│  └────────────────────┘ │                              └────────────────────────┘
└────────────────────────┘
```

---

<p align="center">
  <img src="../images/image_bba957.png" alt="Architecture du Laboratoire SOC" width="700">
</p>

## 🏗️ Partie 1 — Préparation du centre de commande (Ubuntu Server)

### 1.1 Mise à jour du système et outils de base

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install software-properties-common -y
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible -y
```

**Pourquoi ces commandes ?**
- `apt update && apt upgrade` garantit que le système dispose des derniers correctifs de sécurité avant toute installation.
- `software-properties-common` fournit les outils nécessaires pour gérer des dépôts tiers (PPA).
- **Ansible** est installé en prévision d'une future automatisation des tâches de configuration (non utilisé dans ce rapport, mais intégré dans l'environnement de travail).

### 1.2 Première tentative d'installation native de Wazuh

Une première installation a été tentée **directement sur le système hôte** (sans conteneur), en suivant la procédure officielle de Wazuh :

```bash
# 1. Nettoyage préalable de l'environnement (purge de sécurité avant installation)
sudo apt-get purge wazuh-* -y
sudo rm -rf /etc/apt/sources.list.d/wazuh.list /etc/wazuh-* /var/ossec /var/lib/wazuh-*
sudo apt-get autoremove -y

# 2. Import de la clé GPG officielle (vérification d'authenticité des paquets)
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg

# 3. Ajout du dépôt officiel Wazuh
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list

# 4. Mise à jour des index de paquets
sudo apt-get update

# 5. Installation de la stack complète Wazuh
sudo apt-get install wazuh-manager wazuh-indexer wazuh-dashboard -y
```

**Justification du choix initial :** l'installation native a été tentée en premier car elle constitue la méthode "officielle" recommandée par la documentation Wazuh, sans dépendance à une couche de virtualisation supplémentaire.

**⚠️ Résultat — Échec :** l'installation s'est interrompue précisément au moment d'installer **`wazuh-dashboard`**, faute **d'espace disque suffisant** sur le volume logique principal. Le disque virtuel de la VM n'avait pas été étendu après sa création, limitant l'espace réellement disponible pour les index et les fichiers de configuration.

### 1.3 Résolution — Extension de l'espace disque (LVM)

Diagnostic puis extension du volume logique, **à chaud** (sans redémarrage ni interruption de service) :

```bash
# Vérification de l'état des disques et volumes
lsblk

# Extension de l'espace physique (Physical Volume)
sudo pvresize /dev/sda3

# Extension du volume logique (Logical Volume) sur 100% de l'espace libre
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv

# Extension du système de fichiers pour exploiter le nouvel espace
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv

# Vérification finale de l'espace disponible
df -h
```

**Justification technique :**
- `pvresize` permet de signaler au gestionnaire LVM que l'espace physique du disque virtuel a augmenté (suite à un agrandissement du disque dans Hyper-V).
- `lvextend` étend le volume logique sur la totalité de l'espace libre désormais disponible.
- `resize2fs` redimensionne le système de fichiers EXT4 **sans démonter le disque**, garantissant zéro interruption de service.
- Cette approche (LVM) a été choisie plutôt qu'une recréation de VM, car elle est **non destructive** et conserve l'intégralité des données déjà présentes.

### 1.4 Deuxième tentative — Réinstallation complète de Wazuh à zéro

Une fois l'espace disque corrigé, l'installation native a été **relancée depuis zéro** (purge, clé GPG, dépôt, installation — mêmes commandes qu'en 1.2).

**⚠️ Résultat — Nouvel échec :** cette fois, le blocage n'était plus lié à l'espace disque, mais à des **conflits de certificats SSL/TLS**. La version récente d'Ubuntu Server embarque des bibliothèques OpenSSL plus strictes, incompatibles avec certains paquets/scripts d'installation natifs de Wazuh à ce moment-là, rendant le déploiement instable et inabouti.

**Décision technique :** plutôt que de chercher à contourner ces incompatibilités de version (solution fragile et non pérenne), il a été choisi d'abandonner l'installation native et de **migrer vers une solution conteneurisée (Docker)**, qui isole Wazuh de l'environnement système et de ses propres dépendances.

### 1.5 Nettoyage complet du système (Clean Slate)

Avant de basculer vers Docker, un nettoyage rigoureux a été effectué pour repartir sur une base totalement saine.

**Arrêt et désactivation des services :**

```bash
sudo systemctl stop wazuh-manager wazuh-indexer wazuh-dashboard
sudo systemctl disable wazuh-manager wazuh-indexer wazuh-dashboard
```

**Purge des paquets et de leurs fichiers de configuration :**

```bash
sudo apt-get purge -y wazuh-manager wazuh-indexer wazuh-dashboard
sudo apt-get autoremove -y
sudo apt-get autoclean
```

**Nettoyage en profondeur** — suppression des résidus que `apt` ne gère pas (fichiers créés manuellement lors de l'installation) :

```bash
sudo rm -rf /var/ossec
sudo rm -rf /etc/wazuh-*
sudo rm -rf /var/lib/wazuh-*
sudo rm -rf /var/log/wazuh-*
sudo rm -rf /usr/share/wazuh-*
sudo rm -rf /etc/apt/sources.list.d/wazuh.list
sudo rm -rf /usr/share/keyrings/wazuh.gpg
```

**Suppression des comptes système résiduels** (bonne pratique pour éviter des permissions orphelines) :

```bash
sudo deluser wazuh-indexer
sudo deluser wazuh-manager
sudo delgroup wazuh
```

**Vérification finale** (aucune sortie ne doit apparaître si le nettoyage est réussi) :

```bash
ls /etc/ | grep wazuh
```

> ✅ **Résultat :** aucune trace de l'ancienne installation native. Le système est prêt pour une approche conteneurisée propre et reproductible.

---

## 🐳 Partie 2 — Migration vers Docker (solution stable et professionnelle)

**Justification du choix Docker :** après deux échecs natifs (espace disque, puis conflits de certificats), Docker a été retenu car il :
- **isole** Wazuh dans un environnement contrôlé, indépendant des bibliothèques système d'Ubuntu ;
- **simplifie le déploiement** (une stack complète en quelques commandes via Docker Compose) ;
- **facilite la maintenance** (mise à jour, arrêt, redémarrage et suppression sans impacter l'hôte) ;
- correspond à l'**approche professionnelle standard** utilisée en production pour ce type de stack SIEM.

### 2.1 Installation des prérequis système

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
```

Ces paquets permettent au système de communiquer de façon sécurisée (HTTPS) avec les dépôts Docker.

### 2.2 Ajout du dépôt officiel Docker

```bash
# Ajout de la clé GPG officielle de Docker (garantit l'authenticité des paquets)
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Ajout du dépôt officiel, adapté automatiquement à l'architecture et à la version Ubuntu
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 2.3 Installation de Docker Engine et du plugin Compose

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
```

| Paquet | Rôle |
|---|---|
| `docker-ce` | Moteur Docker (Community Edition) |
| `docker-ce-cli` | Interface en ligne de commande |
| `containerd.io` | Runtime bas niveau de gestion des conteneurs |
| `docker-compose-plugin` | Permet d'utiliser `docker compose` (syntaxe moderne, intégrée au CLI) |

### 2.4 Vérification de l'installation

```bash
docker --version
docker compose version
```

### 2.5 Gestion des permissions

```bash
sudo usermod -aG docker $USER
# Une reconnexion (logout/login) est nécessaire pour activer ce changement de groupe
```

**Justification :** ajouter l'utilisateur courant au groupe `docker` évite de devoir préfixer chaque commande par `sudo`, tout en gardant une séparation claire des privilèges (l'utilisateur n'a pas besoin d'être root pour gérer les conteneurs).

### 2.6 Déploiement de la stack Wazuh via Docker Compose

```bash
# Clonage du dépôt officiel Wazuh, sur une version stable et figée (v4.14.0)
git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.0
cd wazuh-docker/single-node

# Génération des certificats de chiffrement pour l'Indexer (TLS interne)
sudo docker compose -f generate-indexer-certs.yml run --rm generator

# Démarrage de l'ensemble des conteneurs (Manager, Indexer, Dashboard) en arrière-plan
sudo docker compose up -d

# Vérification que tous les services sont bien à l'état "Up"
sudo docker compose ps
```

**Justification des choix :**
- L'option `-b v4.14.0` **fige la version** utilisée, garantissant la reproductibilité du lab (évite les ruptures de compatibilité liées à une mise à jour automatique).
- Le mode `single-node` est adapté à un environnement de laboratoire (par opposition à un cluster multi-nœuds destiné à la production).
- La génération des certificats **avant** le démarrage est une étape obligatoire : Wazuh Indexer (basé sur OpenSearch) exige des certificats TLS valides pour établir la communication chiffrée entre ses composants internes.

### 2.7 Commandes d'exploitation courantes

| Action | Commande |
|---|---|
| Démarrer la stack | `cd ~/wazuh-docker/single-node && sudo docker compose up -d` |
| Vérifier l'état des conteneurs | `sudo docker compose ps` |
| Arrêter la stack | `cd ~/wazuh-docker/single-node && sudo docker compose down` |

> ✅ **Résultat :** Wazuh Manager, Indexer et Dashboard sont opérationnels dans des conteneurs Docker isolés, stables, et redéployables en une seule commande.

---

## 🎯 Partie 3 — Préparation de la cible (Kali Linux)

### 3.1 Installation de l'agent Wazuh sur Kali Linux

```bash
# Import de la clé GPG officielle Wazuh
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg

# Ajout du dépôt Wazuh
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee -a /etc/apt/sources.list.d/wazuh.list

# Mise à jour et installation de l'agent
sudo apt update
sudo apt install wazuh-agent -y
```

**Justification technique :** contrairement au Manager (qui nécessite une stack complète Indexer/Dashboard), la machine cible n'a besoin que de **l'agent**, un composant léger chargé de collecter les logs système, les événements de sécurité et l'état d'intégrité des fichiers, puis de les transmettre au Manager pour analyse centralisée.

---

## 🔐 Partie 4 — Liaison sécurisée entre Ubuntu Server et Kali Linux (SSH)

### 4.1 Génération de la paire de clés RSA (sur Ubuntu Server)

```bash
ssh-keygen -t rsa -b 4096
```

Lors des invites, valider chaque étape avec **Entrée** (aucune phrase de passe définie, usage en lab interne) :
- `Enter file in which to save the key` → Entrée
- `Enter passphrase` → Entrée
- `Enter same passphrase again` → Entrée

**Justification :** une clé RSA de **4096 bits** offre un niveau de robustesse cryptographique élevé, supérieur au standard 2048 bits, pour un coût de calcul négligeable dans ce contexte.

### 4.2 Copie de la clé publique vers Kali Linux

```bash
ssh-copy-id kali@172.27.172.52
```

- `Are you sure you want to continue connecting?` → taper **yes**
- `password:` → saisir le mot de passe de l'utilisateur `kali`

**Justification technique :** cette opération établit un **pont chiffré (encrypted bridge)** entre le centre de commande et la cible, permettant l'authentification et la prise de contrôle à distance **sans mot de passe humain** à chaque connexion — ce qui réduit le risque d'interception, de force brute, et permet l'automatisation de tâches (audit à distance, scripts).

### 4.3 Vérification de la connexion sans mot de passe

```bash
ssh kali@172.27.172.52
```

**Résultat obtenu :**

```
Linux kali 6.19.14+kali-amd64 #1 SMP PREEMPT_DYNAMIC Kali 6.19.14-1+kali1 (2026-05-05) x86_64
Last login: Sun Jun 28 08:38:38 2026 from 172.27.160.1
┌──(kali㉿kali)-[~]
└─$
```

> ✅ La connexion s'établit **sans demande de mot de passe**, confirmant le bon fonctionnement de l'authentification par clé publique/privée.

---

## 🕵️ Partie 5 — Audit de sécurité avec Lynis

### 5.1 Installation de Lynis sur Kali Linux

```bash
sudo apt update
sudo apt install lynis -y
```

**Justification du choix de l'outil :** Lynis est un outil d'audit open-source reconnu, capable d'analyser la configuration système (durcissement, permissions, services, réseau, comptes) et de fournir des recommandations classées par sévérité — un complément pertinent à la supervision en temps réel assurée par Wazuh.

### 5.2 Premier audit (exécution locale, complète)

```bash
sudo lynis audit system
```

### 5.3 Audit à distance depuis Ubuntu Server (Fleet Management)

```bash
ssh kali@172.27.172.52 "sudo lynis audit system --quick"
```

**Justification technique :** déclencher l'audit à distance, depuis le centre de commande, permet une **gestion centralisée de la flotte (Fleet Management)** — c'est-à-dire piloter l'audit de plusieurs machines depuis un seul poste, sans avoir à se connecter physiquement à chaque cible. L'option `--quick` accélère l'exécution en réduisant les vérifications les plus longues, adapté à un contrôle de routine.

### 5.4 Export du rapport d'audit complet

Exécution locale avec sortie redirigée :

```bash
sudo lynis audit system > ~/rapport_securite_kali.txt
```

Ou, déclenchée à distance avec sortie redirigée et conservée sur Ubuntu Server (pseudo-terminal forcé avec `-t` pour préserver l'affichage interactif de Lynis) :

```bash
ssh -t kali@172.27.172.52 "sudo lynis audit system --quick" > rapport_securite_kali.txt
```

---

## 🔧 Partie 6 — Mesures correctives appliquées (Remediation)

L'analyse du rapport Lynis a fait apparaître deux avertissements (`WARNING`), corrigés comme suit :

### 6.1 Avertissement — Permissions du dossier `/etc/sudoers.d`

**Constat Lynis :**
```
Permissions for directory: /etc/sudoers.d [ WARNING ]
```

**Correction appliquée :**

```bash
sudo chown -R root:root /etc/sudoers.d
sudo chmod 755 /etc/sudoers.d
```

**Justification technique :** le dossier `/etc/sudoers.d` contient des fichiers définissant les privilèges d'exécution `sudo`. Restreindre sa propriété au seul compte `root` et limiter ses permissions à `755` empêche toute modification non autorisée par un autre utilisateur ou processus, réduisant ainsi le risque d'**escalade de privilèges** (un utilisateur non autorisé qui parviendrait à modifier ce dossier pourrait s'octroyer des droits root).

### 6.2 Avertissement — Résolution DNS insuffisante

**Constat Lynis :**
```
Minimal of 2 responsive nameservers [ WARNING ]
```

**Correction appliquée :**

```bash
sudo nano /etc/resolv.conf
```
*(ajout d'un second serveur DNS fonctionnel dans le fichier)*

**Justification technique :** disposer d'**au moins deux serveurs DNS** opérationnels garantit la continuité de la résolution de noms en cas de panne ou d'indisponibilité du premier serveur. Cela limite également l'exposition à certaines attaques basées sur le DNS (spoofing, déni de service par dépendance à un unique résolveur).

---

## 📊 Tableau récapitulatif des problèmes et solutions

| # | Problème rencontré | Cause racine | Solution appliquée | Résultat |
|---|---|---|---|---|
| 1 | Échec installation Wazuh (1ʳᵉ tentative, bloqué sur `wazuh-dashboard`) | Espace disque insuffisant sur le volume LVM | Extension LVM (`pvresize`, `lvextend`, `resize2fs`) | ✅ Espace disque corrigé |
| 2 | Échec installation Wazuh (2ᵉ tentative, native) | Conflits de certificats SSL/TLS liés à une version récente d'Ubuntu | Purge complète du système + migration vers Docker | ✅ Stack stable et isolée |
| 3 | Permissions `/etc/sudoers.d` trop larges | Configuration par défaut du système | `chown root:root` + `chmod 755` | ✅ Risque d'escalade de privilèges réduit |
| 4 | DNS instable / nameservers insuffisants | Configuration réseau incomplète | Ajout d'un second résolveur dans `/etc/resolv.conf` | ✅ Résolution DNS fiabilisée |

---

## 📜 Conclusion

Ce laboratoire SOC, désormais pleinement opérationnel, repose sur les fondations suivantes :

- Un **Wazuh Manager / Indexer / Dashboard** stable, déployé sous **Docker**, après l'échec de deux tentatives d'installation native (espace disque insuffisant, puis conflits de certificats liés à la version d'Ubuntu) ;
- Une **cible Kali Linux** équipée de l'agent Wazuh, supervisée en temps réel depuis le Manager ;
- Une **liaison SSH chiffrée** par clé RSA 4096 bits, sans authentification par mot de passe, permettant un pilotage centralisé (Fleet Management) ;
- Un **audit de sécurité Lynis**, ayant permis d'identifier et de corriger deux vulnérabilités de configuration (permissions `sudoers`, résolution DNS).

Chaque difficulté technique rencontrée au cours du projet — manque d'espace disque, incompatibilités de certificats — a été traitée par un diagnostic ciblé suivi d'une solution durable, plutôt qu'un simple contournement temporaire. Cette démarche reflète une approche d'ingénierie système rigoureuse, transposable à un contexte professionnel de mise en place d'un SOC.

Ce laboratoire constitue désormais une base solide pour des évolutions futures : configuration de règles d'alertes personnalisées (decoders/rules Wazuh), intégration d'un module de réponse automatisée aux incidents (SOAR), ou ajout de cibles supplémentaires au sein de la flotte supervisée.

---

*Projet réalisé par Youssef — 2026*
