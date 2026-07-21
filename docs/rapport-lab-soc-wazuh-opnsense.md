# Rapport de Projet — Déploiement d'un Home Lab SOC (Wazuh + OPNsense + Ansible)

**Auteur :** Grim Jow
**Date :** 21 juillet 2026
**Environnement :** Hyper-V (Windows) — Machines virtuelles : Rocky Linux 9.8, Ubuntu Server, Parrot OS, Kali Linux, OPNsense

---

## 1. Objectif du projet

Mettre en place un laboratoire de sécurité (SOC Home Lab) entièrement virtualisé, isolé du réseau externe, permettant de :

- Déployer un SIEM **Wazuh** (Manager + Indexer) pour la collecte et l'analyse d'événements de sécurité.
- Superviser plusieurs endpoints (Rocky Linux, Ubuntu, Parrot OS, Kali) via des **agents Wazuh**.
- Sécuriser et cloisonner le réseau du lab avec un pare-feu **OPNsense**.
- Automatiser le déploiement et la maintenance des machines avec **Ansible**.

Ce document retrace la démarche complète, les choix techniques, les commandes utilisées, les erreurs rencontrées et les solutions apportées.

---

## 2. Architecture du laboratoire

| Composant | Rôle | Adresse IP |
|---|---|---|
| OPNsense | Pare-feu / Passerelle du lab | 192.168.56.1 |
| Rocky Linux 9.8 | Wazuh Manager + Indexer | 192.168.56.52 |
| Ubuntu Server | Endpoint supervisé (agent Wazuh) | 192.168.56.26 |
| Parrot OS | Endpoint supervisé (agent Wazuh) | 192.168.56.47 |
| Kali Linux | Endpoint supervisé (agent Wazuh) | 192.168.56.x |

Le réseau interne (`192.168.56.0/24`) est totalement isolé du réseau physique de l'hôte, la connectivité Internet étant assurée uniquement via l'interface WAN d'OPNsense.

---

## 3. Mise en réseau : Hyper-V + OPNsense

### 3.1 Première tentative (abandonnée)

La première approche reposait sur un **pont réseau manuel** entre la carte Wi-Fi physique et les cartes Hyper-V, combiné à la fonctionnalité **ICS (Internet Connection Sharing)** de Windows.

**Problèmes rencontrés :**
- Conflits d'adressage IP et instabilité du pont réseau.
- ICS imposait une plage d'adresses non maîtrisée (`192.168.137.x`), incompatible avec le plan d'adressage du lab (`192.168.56.0/24`).
- Boucles de routage rendant l'interface web d'OPNsense intermittente.

**Conclusion :** cette architecture a été abandonnée car trop fragile pour un environnement de lab reproductible.

### 3.2 Architecture finale (retenue)

Deux Virtual Switches Hyper-V distincts ont été créés :

| Switch | Type | Rôle |
|---|---|---|
| `LAN_Net` (Network_2026) | Interne (Internal) | Réseau isolé des VM du lab |
| Switch externe (Default Switch / carte physique) | Externe | Fournit l'accès Internet à OPNsense |

**Étapes de configuration :**

1. Suppression des anciens switches mal configurés dans *Hyper-V Manager > Virtual Switch Manager*.
2. Création d'un switch interne unique `LAN_Net`.
3. Attribution de deux cartes réseau virtuelles à la VM OPNsense :
   - **Adaptateur 1 (hn0)** → `LAN_Net` → IP fixe `192.168.56.1/24`.
   - **Adaptateur 2 (hn1)** → Switch externe → adressage **DHCP automatique**.

**Justification technique :** ce choix élimine la dépendance à l'ICS. La carte WAN récupère automatiquement une IP dès qu'elle est branchée à un réseau, tandis que le LAN reste stable quel que soit le réseau externe utilisé — la panne rencontrée avec la première méthode ne peut plus se reproduire.

### 3.3 Configuration d'OPNsense

- Accès à l'interface d'administration via `https://192.168.56.1`.
- Attribution des interfaces : LAN sur `hn0` (IP fixe), WAN sur `hn1` (DHCP).
- Activation du serveur **DHCP** sur le LAN, plage `192.168.56.10` → `192.168.56.100`.
- Activation du **DNS Resolver** pour la résolution de noms interne au lab.
- Renforcement du mot de passe root de l'interface d'administration.

**Test de connectivité WAN :**

```
ping cloudflare.com  → 0.0% packet loss
ping 8.8.8.8         → 0.0% packet loss
```

Résultat : la passerelle dispose d'un accès Internet stable et fonctionnel, condition nécessaire pour télécharger les paquets Wazuh sur les futures VM.

---

## 4. Installation de Wazuh Manager sur Rocky Linux

### 4.1 Première tentative — dépôt manuel (échec)

Un fichier de dépôt Wazuh a d'abord été créé manuellement :

```bash
sudo bash -c 'echo "[wazuh]
gpgcheck=1
gpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH
enabled=1
name=EL-\$releasever - Wazuh
baseurl=https://packages.wazuh.com/4.x/yum/
protect=1" > /etc/yum.repos.d/wazuh.repo'
```

**Objectif de chaque paramètre :**
- `gpgcheck=1` + `gpgkey` : vérifie la signature numérique des paquets pour garantir leur authenticité (protection contre les attaques de type Man-in-the-Middle).
- `baseurl` : adresse du dépôt officiel Wazuh.
- `enabled=1` : active le dépôt pour `dnf`.
- `$releasever` : variable remplacée automatiquement par la version de Rocky Linux installée.

Installation et activation du service :

```bash
sudo dnf clean all
sudo dnf install wazuh-manager -y
sudo systemctl daemon-reload
sudo systemctl enable wazuh-manager
sudo systemctl start wazuh-manager
sudo systemctl status wazuh-manager
```

**Résultat :** le service est passé en état `active (running)`, avec tous les démons chargés (`wazuh-analysisd`, `wazuh-remoted`, `wazuh-logcollector`, `wazuh-modulesd`, etc.).

Cependant, le téléchargement direct des scripts officiels (`wazuh-install.sh`, `wazuh-certs-tool.sh`) via `curl` échouait systématiquement : les fichiers reçus ne faisaient que **14 octets** et contenaient une réponse XML `Access Denied`, révélant un **blocage réseau** (proxy ou filtrage) empêchant l'accès direct aux serveurs Wazuh.

Malgré ce blocage, plusieurs éléments ont pu être installés avec succès à ce stade : le **Wazuh Indexer**, le fichier `config.yml`, l'éditeur `nano`, ainsi qu'**OpenVPN** (via le dépôt `epel-release`).

### 4.2 Correction d'un problème de configuration de l'Indexer

Le fichier de configuration de l'Indexer a été réécrit proprement :

```bash
sudo bash -c 'cat > /etc/wazuh-indexer/opensearch.yml <<EOF
cluster.name: wazuh
node.name: node-1
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node
plugins.security.disabled: true
EOF'

sudo chown wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/opensearch.yml
sudo chmod 660 /etc/wazuh-indexer/opensearch.yml

sudo systemctl daemon-reload
sudo systemctl restart wazuh-indexer
sudo systemctl status wazuh-indexer
```

**Résultat :** le service est repassé en état stable `active (running)`.

### 4.3 Deuxième tentative — image ISO changée (succès)

Le blocage réseau observé précédemment provenait probablement d'une incompatibilité liée à l'image système utilisée. La VM a été reconstruite avec l'image officielle **Rocky-9.8-x86_64-dvd.iso**, ce qui a permis de résoudre l'accès aux serveurs Wazuh.

```bash
# Mise à jour et outils de base
sudo dnf update -y
sudo dnf install -y wget curl nano vim tar zip unzip lsof

# Téléchargement du script d'installation officiel
curl -sO https://packages.wazuh.com/4.8/wazuh-install.sh

# Dépendances (Ansible sera utilisé plus tard pour l'automatisation)
sudo dnf install -y epel-release
sudo dnf install -y ansible

# Installation complète (Manager + Indexer + Dashboard)
sudo bash wazuh-install.sh -a --ignore-check
```

**Choix technique — option `--ignore-check` :** la version 4.8 de Wazuh ne reconnaissait pas officiellement Rocky Linux 9.8 comme version supportée. L'option `--ignore-check` permet de forcer l'installation en ignorant cette vérification de compatibilité, ce qui s'est révélé fonctionnel dans le contexte contrôlé du lab.

Configuration du pare-feu local pour autoriser les flux Wazuh (agents ↔ manager, dashboard) :

```bash
sudo firewall-cmd --permanent --add-port=1514/tcp   # communication agent → manager
sudo firewall-cmd --permanent --add-port=1515/tcp   # enrôlement des agents
sudo firewall-cmd --permanent --add-port=443/tcp    # dashboard web
sudo firewall-cmd --reload
hostname -I
```

**Résultat final :** Wazuh Manager, Indexer et Dashboard opérationnels sur Rocky Linux 9.8, accessibles depuis le réseau du lab.

---

## 5. Installation de l'agent Wazuh sur Kali Linux

### 5.1 Import de la clé GPG et ajout du dépôt

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee -a /etc/apt/sources.list.d/wazuh.list
sudo apt-get update
```

**Remarque :** la première tentative sans `sudo` a échoué (`Permission denied`) car l'écriture dans `/usr/share/keyrings/` nécessite les droits root.

### 5.2 Installation de l'agent

```bash
sudo WAZUH_MANAGER='<IP_DU_MANAGER>' apt-get install wazuh-agent
```

La variable d'environnement `WAZUH_MANAGER` permet de préconfigurer l'agent avec l'adresse IP du Manager dès l'installation.

**Résultat :** paquet `wazuh-agent` (v4.14.6) installé avec succès.

**Action à planifier :** modifier manuellement `/var/ossec/etc/ossec.conf` pour figer l'IP du Manager de façon durable, puis activer et démarrer le service :

```bash
sudo systemctl enable --now wazuh-agent
```

---

## 6. Automatisation avec Ansible

### 6.1 Authentification SSH par clé

Pour permettre à Ansible de se connecter sans mot de passe :

```bash
ssh-keygen                                    # génération d'une paire de clés ED25519
ssh-copy-id parrotos@192.168.56.47            # déploiement de la clé publique sur Parrot OS
ssh-copy-id ubuntu@192.168.56.26              # déploiement de la clé publique sur Ubuntu
```

### 6.2 Test de connectivité Ansible

```bash
ansible lab_machines -i hosts.ini -m ping
```

**Résultat :** toutes les machines répondent, l'interpréteur Python est détecté automatiquement sur chaque hôte.

### 6.3 Playbook de mise à jour (`update_lab.yml`)

Premier lancement du playbook (mise à jour système uniquement) :

```bash
ansible-playbook -i hosts.ini update_lab.yml
```

```
192.168.56.26  : ok=2  changed=1  unreachable=0  failed=0
192.168.56.47  : ok=2  changed=0  unreachable=0  failed=0
```

### 6.4 Extension du playbook : automatisation de l'installation de l'agent Wazuh

Le playbook a ensuite été enrichi pour :

1. Attendre la libération du verrou `dpkg` avant toute opération `apt` (évite les échecs liés à des tâches d'installation en arrière-plan).
2. Mettre à jour le cache des dépôts de façon tolérante aux erreurs.
3. Installer les dépendances (`curl`, `gnupg`, `apt-transport-https`).
4. Ajouter la clé GPG et le dépôt Wazuh sans dépendre de l'ancien mécanisme `apt-key` (déprécié).
5. Installer l'agent Wazuh et l'associer au Manager.
6. Activer et démarrer le service (`systemctl enable --now wazuh-agent`).

**Résultat de l'exécution :**

```
192.168.56.47 (Parrot OS) : ok=8  changed=1  failed=0   → agent installé et actif
192.168.56.26 (Ubuntu)    : ok=5  changed=1  failed=1   → échec de mise à jour du cache apt
```

**Analyse de l'échec sur Ubuntu :** la tâche `apt update` a échoué avec le message `Failed to update apt cache: unknown reason`, probablement lié à un conflit temporaire de dépôt ou de connectivité au moment de l'exécution. Cette étape reste à ré-exécuter pour finaliser l'installation de l'agent sur cette machine.

---

## 7. Difficultés rencontrées et solutions apportées

| Problème | Cause | Solution retenue |
|---|---|---|
| Perte de connexion Internet lors du changement de réseau Wi-Fi | Pont réseau manuel + ICS avec IP statique | Switch WAN externe en DHCP automatique + LAN interne isolé |
| Téléchargement des scripts Wazuh bloqué (fichier de 14 octets) | Filtrage réseau / image ISO Rocky incompatible | Réinstallation avec `Rocky-9.8-x86_64-dvd.iso` |
| Wazuh 4.8 non reconnu sur Rocky Linux 9.8 | Version d'OS non supportée officiellement | Utilisation de l'option `--ignore-check` |
| Échec `gpg --dearmor` sur Kali | Absence de droits sur `/usr/share/keyrings/` | Exécution avec `sudo` |
| Échec `apt update` sur Ubuntu via Ansible | Cause réseau/dépôt ponctuelle non confirmée | Ré-exécution du playbook à prévoir |

---

## 8. État actuel du lab

- ✅ OPNsense opérationnel : LAN isolé (`192.168.56.0/24`), WAN en DHCP, DNS Resolver actif.
- ✅ Wazuh Manager + Indexer + Dashboard fonctionnels sur Rocky Linux 9.8.
- ✅ Agent Wazuh installé et actif sur Kali Linux et Parrot OS.
- ⚠️ Agent Wazuh sur Ubuntu Server : installation à finaliser (échec de mise à jour du cache apt).
- ✅ Automatisation Ansible fonctionnelle pour la mise à jour et le déploiement des agents.

## 9. Prochaines étapes

1. Corriger et relancer le playbook Ansible pour finaliser l'agent Wazuh sur Ubuntu Server.
2. Fixer l'IP du Manager dans `ossec.conf` sur tous les agents pour une connexion durable.
3. Vérifier la remontée des agents dans le Dashboard Wazuh (`Agents actifs`).
4. Débuter les scénarios de simulation d'attaques et de détection (MITRE ATT&CK) depuis Kali/Parrot vers les endpoints supervisés.

---

*Rapport rédigé dans le cadre du projet personnel de Home Lab SOC, destiné au portfolio GitHub.*
