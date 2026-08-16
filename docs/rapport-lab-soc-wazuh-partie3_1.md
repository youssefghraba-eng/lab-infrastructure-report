# Rapport de Projet — Home Lab SOC (Wazuh + Ansible + Tailscale) — Partie 3

**Auteur :** Grim Jow
**Date :** 1er août 2026
**Suite de :** *Rapport 1 — Déploiement Wazuh + OPNsense + Ansible* / *Rapport 2 — Enrôlement des agents et incompatibilités de version*

---

## 1. Objectif de cette session

Cette session clôt la phase de **construction du laboratoire** avec trois objectifs :

1. Finaliser et stabiliser la connexion de l'agent Wazuh sur Kali Linux, jusqu'à son affichage en statut **Active** dans le Dashboard.
2. Construire un tunnel **Tailscale** pour contourner l'isolation réseau entre Hyper-V et VMware, condition nécessaire à la communication Kali ↔ Manager et Kali ↔ Ubuntu.
3. Clore la construction du lab (hors Proxmox, non traité dans cette phase) et définir la feuille de route des scénarios d'attaque à venir.

---

## 2. Étape 1 — Tentative de connexion directe de l'agent Kali

### 2.1 Configuration de l'adresse du Manager

```bash
sudo sed -i "s/<ADDRESS>.*<\/ADDRESS>/<ADDRESS>192.168.56.52<\/ADDRESS>/" /var/ossec/etc/ossec.conf
```

Cette commande remplace directement la balise `<ADDRESS>` du fichier `ossec.conf` par l'IP locale du Manager (`192.168.56.52`), sans avoir à ouvrir le fichier manuellement.

### 2.2 Rechargement et redémarrage du service

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl restart wazuh-agent
```

### 2.3 Vérification locale

```bash
sudo systemctl status wazuh-agent
```

**Résultat :** le service est bien `active (running)`, tous les démons (`wazuh-execd`, `wazuh-agentd`, `wazuh-syscheckd`, `wazuh-logcollector`, `wazuh-modulesd`) sont chargés sans erreur, avec la version confirmée : `Starting Wazuh v4.8.2...`.

### 2.4 Constat côté Dashboard : Kali absent

Malgré un service local sain, le Dashboard Wazuh affichait :

| Statut | Nombre |
|---|---|
| Active | 1 |
| Disconnected | 1 |
| Pending | 0 |
| Never connected | 0 |

**Couverture des agents : 50.00 %** — seul l'agent `server` (Ubuntu, v4.8.2) apparaissait comme actif dans la table des agents. Kali Linux restait totalement invisible.

### 2.5 Diagnostic par les journaux

```bash
sudo tail -f /var/ossec/logs/ossec.log
```

**Preuve du blocage (extrait) :**

```
2026/08/01 13:12:41 wazuh-agentd: INFO: Requesting a key from server: 192.168.56.52
2026/08/01 13:12:44 wazuh-agentd: ERROR: (1208): Unable to connect to enrollment service at '[192.168.56.52]:1515'
2026/08/01 13:13:19 wazuh-agentd: INFO: Requesting a key from server: 192.168.56.52
2026/08/01 13:13:23 wazuh-agentd: ERROR: (1208): Unable to connect to enrollment service at '[192.168.56.52]:1515'
```

L'erreur se répète en boucle toutes les ~40 secondes : Kali tente en permanence d'obtenir une clé d'enrôlement auprès du service sur le port **1515**, sans jamais y parvenir.

### 2.6 Analyse de la cause

Ce symptôme confirme le même diagnostic que dans le Rapport 2 : **isolation réseau totale entre l'environnement VMware (Kali) et l'environnement Hyper-V (Rocky Linux/Manager)**. Même si les adresses IP appartiennent au même sous-réseau apparent (`192.168.56.x`), les mécanismes de routage virtuel propres à chaque hyperviseur empêchent le trafic d'enrôlement (port 1515) de circuler entre les deux environnements — contrairement à Ubuntu et Parrot, qui se trouvent tous deux dans l'écosystème Hyper-V et communiquent sans problème.

**Conclusion :** une connexion directe par IP locale est impossible entre Kali (VMware) et le Manager (Hyper-V). Passage à l'étape 2, prévue dans la feuille de route : le tunnel Tailscale.

---

## 3. Étape 2 — Construction du tunnel Tailscale

### 3.1 Installation sur Kali Linux (attaquant)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

```
To authenticate, visit:
        https://login.tailscale.com/a/20a1274018a85

Success.
```

Récupération de l'adresse overlay attribuée à Kali :

```bash
tailscale ip -4
```

```
100.89.2.93
```

### 3.2 Installation sur Rocky Linux (Manager)

```bash
sudo tailscale up
```

```
To authenticate, visit:
        https://login.tailscale.com/a/193d4b20010f53

Success.
```

```bash
tailscale ip -4
```

```
100.124.203.84
```

### 3.3 Bilan des adresses Tailscale (Tailnet du lab)

| Machine | Rôle | IP réseau local | IP Tailscale (overlay) |
|---|---|---|---|
| Rocky Linux | Wazuh Manager | 192.168.56.52 | 100.124.203.84 |
| Kali Linux | Attaquant | 192.168.56.129 (VMware) | 100.89.2.93 |
| Ubuntu Server | Cible | 192.168.56.26 (Hyper-V) | *(installé à l'étape suivante)* |

**Principe :** contrairement à un tunnel WireGuard pair-à-pair (qui nécessite un chemin réseau direct, inexistant ici), Tailscale s'appuie sur des serveurs relais (STUN/TURN) sur Internet pour établir la connexion, ce qui permet de relier deux machines dans des environnements de virtualisation totalement cloisonnés — sans toucher à la configuration des switches virtuels Hyper-V/VMware, et donc sans risquer l'instabilité déjà rencontrée avec les ponts réseau manuels (voir Rapport 2, §7.4).

---

## 4. Étape finale — Redirection de l'agent Wazuh vers l'IP Tailscale

### 4.1 Mise à jour de l'adresse du Manager côté Kali

```bash
sudo sed -i "s/<ADDRESS>.*<\/ADDRESS>/<ADDRESS>100.124.203.84<\/ADDRESS>/" /var/ossec/etc/ossec.conf
```

L'adresse locale (`192.168.56.52`, inaccessible) est remplacée par l'adresse Tailscale du Manager (`100.124.203.84`), désormais joignable via le tunnel overlay.

### 4.2 Redémarrage du service

```bash
sudo systemctl restart wazuh-agent
```

### 4.3 Vérification des journaux — preuve de la connexion réussie

```bash
sudo tail -n 20 /var/ossec/logs/ossec.log
```

```
2026/08/01 13:52:39 wazuh-logcollector: INFO: (1950): Analyzing file: '/var/log/apache2/access.log'.
2026/08/01 13:52:39 wazuh-logcollector: INFO: (1950): Analyzing file: '/var/ossec/logs/active-responses.log'.
2026/08/01 13:52:39 wazuh-logcollector: INFO: (1950): Analyzing file: '/var/log/dpkg.log'.
2026/08/01 13:52:39 wazuh-logcollector: INFO: Started (pid: 32076).
2026/08/01 13:52:39 wazuh-modulesd: INFO: Started (pid: 32091).
2026/08/01 13:52:39 sca: INFO: Starting Security Configuration Assessment scan.
2026/08/01 13:52:39 wazuh-modulesd:syscollector: INFO: Module started.
2026/08/01 13:52:42 sca: INFO: Security Configuration Assessment scan finished. Duration: 3 seconds.
2026/08/01 13:53:11 wazuh-syscheckd: INFO: (6009): File integrity monitoring scan ended.
2026/08/01 13:53:11 wazuh-syscheckd: INFO: FIM sync module started.
```

**Résultat :** disparition totale des erreurs `(1208)`. L'agent charge normalement ses modules (`logcollector`, `modulesd`, `sca`, `syscheckd`) et exécute ses scans habituels (Security Configuration Assessment, File Integrity Monitoring), preuve que la communication avec le Manager est désormais stable via le tunnel Tailscale.

### 4.4 Extension du tunnel à la cible (Ubuntu Server)

Pour permettre également la future communication d'attaque Kali → Ubuntu via l'overlay Tailscale, ce dernier a été installé sur Ubuntu Server (Hyper-V) :

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

**Résultat :** les trois nœuds critiques du lab (Manager, attaquant, cible) sont désormais joignables sur le même Tailnet, indépendamment de leur hyperviseur d'origine.

---

## 5. Synthèse des problèmes et solutions (session du jour)

| Problème | Cause racine | Solution appliquée |
|---|---|---|
| Service Wazuh actif localement sur Kali mais absent du Dashboard | — | Diagnostic par les logs (`ossec.log`) |
| `ERROR: (1208): Unable to connect to enrollment service at [192.168.56.52]:1515` en boucle | Isolation réseau totale entre le switch virtuel VMware (Kali) et le switch virtuel Hyper-V (Manager) | Construction d'un tunnel overlay Tailscale entre les deux machines |
| Choix du type de VPN | Un tunnel WireGuard pair-à-pair nécessite un chemin réseau direct, inexistant ici | Utilisation de Tailscale (relais STUN/TURN) plutôt qu'un pont réseau manuel, pour préserver la stabilité du lab |
| Redirection du trafic Wazuh vers le tunnel | `ossec.conf` pointait vers l'IP locale inaccessible | Remplacement de `<ADDRESS>` par l'IP Tailscale du Manager (`100.124.203.84`) puis redémarrage du service |

---

## 6. État final de la construction du laboratoire

| Composant | Statut |
|---|---|
| OPNsense (pare-feu / passerelle du lab) | ✅ Opérationnel |
| Wazuh Manager (Rocky Linux, v4.8.2) | ✅ Opérationnel |
| Agent Wazuh — Ubuntu Server | ✅ Actif et connecté |
| Agent Wazuh — Parrot OS | ✅ Actif et connecté |
| Agent Wazuh — Kali Linux | ✅ Actif et connecté (via tunnel Tailscale) |
| Automatisation Ansible (mise à jour, déploiement des agents) | ✅ Fonctionnelle |
| Tunnel Tailscale (Kali ↔ Manager ↔ Ubuntu) | ✅ Opérationnel |
| Proxmox | ⏸️ Non traité — rôle prévu ultérieurement |

**Bilan :** la phase de **construction du laboratoire est achevée**. Le SOC Home Lab dispose désormais d'une supervision centralisée (Wazuh), d'un pare-feu périmétrique (OPNsense), d'une automatisation de déploiement (Ansible) et d'une connectivité inter-hyperviseurs fiable (Tailscale). Le projet entre maintenant dans sa phase de **démonstration offensive/défensive**.

---

## 7. Feuille de route des scénarios d'attaque (phase suivante)

Cette section documente le plan de tests prévu pour la suite du projet : chaque famille d'attaque est associée à l'outil d'exécution (côté Kali), au mécanisme de détection (Wazuh / OPNsense), et à la contre-mesure de défense recommandée.

### 7.1 Attaques réseau et couche 2 (Layer 2 & Network Attacks)

**ARP Spoofing / MITM** (via Bettercap / Ettercap)
- *Détection :* OPNsense surveille les modifications de la table ARP et les émissions d'ARP gratuit (Gratuitous ARP) suspectes ; Wazuh capture les journaux anormaux associés.
- *Analyse :* observation dans Wireshark de réponses ARP (ARP Reply) répétées et suspectes.
- *Défense :* activation de la Dynamic ARP Inspection (DAI) et du DHCP Snooping sur le switch ou le pare-feu.

**DHCP Starvation & Rogue DHCP**
- *Détection :* Wazuh surveille l'apparition d'appareils non autorisés sur le réseau.
- *Défense :* activation du DHCP Snooping avec définition des ports de confiance et non fiables.

### 7.2 Attaques par déni de service (DoS / DDoS & Floods)

**SYN Flood / UDP Flood** (via Hping3)
- *Détection :* pic soudain et anormal des alertes réseau dans Wazuh, confirmé par une analyse de trafic en direct dans Wireshark.
- *Défense :* activation des SYN Cookies et des limiteurs de débit (Rate Limiting) dans OPNsense.

### 7.3 Attaques web (XSS, injection SQL, exécution de code à distance)

**Exploitation via Sqlmap, Burp Suite, Metasploit**
- *Exécution :* tentatives d'injection de base de données ou exploitation de vulnérabilités depuis Kali.
- *Détection :* Wazuh surveille les journaux d'accès et d'erreurs (Access/Error Logs) des applications web (Apache/Nginx) et capture les requêtes suspectes.
- *Défense :* mise en place d'un WAF (Web Application Firewall), validation stricte des entrées, encodage des sorties.

### 7.4 Malware et post-exploitation

**Metasploit / Meterpreter / Rootkits**
- *Exécution :* génération de payload et test de connexion inversée (reverse shell).
- *Détection (HIDS) :* Wazuh surveille l'intégrité des fichiers (File Integrity Monitoring) et détecte toute modification de fichiers système.
- *Outils d'audit complémentaires :* Lynis pour l'audit de sécurité Linux, chkrootkit et rkhunter pour la détection de rootkits.
- *Défense :* gestion continue des correctifs (patch management), pare-feu robuste, activation de la protection au niveau hôte (HIDS/HIPS).

### 7.5 Attaques Wi-Fi / sans fil

**Evil Twin & Deauthentication** (via Aircrack-ng / Airgeddon)
- *Détection :* surveillance des points d'accès frauduleux diffusant le même SSID.
- *Défense :* adoption du standard WPA3 et activation d'un système de détection d'intrusion sans fil (WIPS).

---

## 8. Prochaine étape immédiate

Démarrage du **premier scénario offensif pratique : ARP Spoofing / MITM**, exécuté depuis Kali Linux, analysé en direct via Wireshark, et observé côté détection via le Wazuh Dashboard. Ce scénario servira de première validation end-to-end de la chaîne complète construite dans les trois rapports précédents : *attaque → capture réseau → détection SIEM*.

---

*Rapport rédigé dans le cadre du projet personnel de Home Lab SOC, destiné au portfolio GitHub — suite directe des Rapports 1 et 2, clôture de la phase de construction du laboratoire.*
