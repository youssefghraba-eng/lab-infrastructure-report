[README.md](https://github.com/user-attachments/files/31357361/README.md)
# 🛡️ Portfolio Cybersécurité — Youssef Ghraba

![Status](https://img.shields.io/badge/status-actif-success)
![Focus](https://img.shields.io/badge/focus-SOC%20%7C%20Red%20%26%20Blue%20Team-critical)
![Stack](https://img.shields.io/badge/stack-Proxmox%20%7C%20Hyper--V%20%7C%20OPNsense%20%7C%20Wazuh-informational)

Ce dépôt regroupe l'ensemble de mes projets de laboratoire en cybersécurité, menés dans le cadre de ma spécialisation en ingénierie de la sécurité et des opérations (SecOps / SOC Analyst / Blue & Red Team). Chaque projet est documenté intégralement et conservé dans ce portfolio afin de tracer l'évolution de mon apprentissage et de mes compétences techniques.

## 📑 Sommaire des Projets

| # | Projet | Statut | Description courte |
|---|---|---|---|
| 1 | [**Enterprise SOC Lab 2026**](#-projet-2--enterprise-soc-lab-2026--laboratoire-professionnel-doperations-de-sécurité) | 🟡 En cours | Laboratoire SOC professionnel avec SIEM, SOAR, scénarios TDIR complets |
| 2 | [**Cybersecurity Lab Infrastructure**](#-projet-1--cybersecurity-lab-infrastructure--dual-perspective-portfolio-red--blue-teaming) | 🟢 Réalisé | Laboratoire fondateur Red & Blue Teaming, TTPs offensives et défensives |

---
---

# 🎯 Projet 2 — Enterprise SOC Lab 2026 — Laboratoire Professionnel d'Opérations de Sécurité

![Status](https://img.shields.io/badge/status-en%20cours-yellow)
![Focus](https://img.shields.io/badge/focus-SOC%20%7C%20Red%20%26%20Blue%20Team-critical)
![Stack](https://img.shields.io/badge/stack-Proxmox%20%7C%20Hyper--V%20%7C%20OPNsense%20%7C%20Wazuh-informational)

## 📌 Introduction & Overview

Ce projet documente la conception, la construction et l'exploitation d'un **laboratoire professionnel d'opérations de sécurité (SOC)**, pensé pour reproduire les conditions réelles d'un centre d'opérations de sécurité en entreprise.

L'objectif est double :

1. **Construire une infrastructure segmentée réaliste**, incluant zones réseau, services exposés et cibles vulnérables.
2. **Dérouler des scénarios d'attaque et de défense complets**, en suivant le cycle de vie complet d'un incident de sécurité : détection, investigation, confinement et remédiation (méthodologie **TDIR** — *Threat Detection, Investigation & Response*).

Chaque scénario est documenté selon une structure rigoureuse et reproductible, dans une logique de formation continue en SecOps / Blue & Red Team.

## 🏗️ 1. Architecture & Segmentation Réseau (Network Zoning & Topology)

Le laboratoire s'appuie sur l'intégration de deux plateformes de virtualisation — **Proxmox VE** (Giada Mini PC) et **Hyper-V** (Panasonic Notebook) — afin de simuler une infrastructure d'entreprise réaliste, segmentée en VLANs via le pare-feu **OPNsense**.

| Zone | Rôle | Composants |
|---|---|---|
| **LAN (Internal Trusted Zone)** | Zone interne de confiance | Postes du Blue Team, machines victimes (ex. `Ubuntu_Victim_Server` avec agent Wazuh) |
| **DMZ (Demilitarized Zone)** | Services exposés | Serveurs web, honeypots, interfaces publiques |
| **WLAN / Guest Zone** | Réseau isolé | Tests et exploitation de réseaux sans fil |
| **WAN / External** | Simulation Internet | Interface de connexion principale |

## 🛠️ 2. Arsenal d'Outils de l'Analyste SOC (2026 Toolkit)

### Analyse réseau & analyse web
- **Wireshark / Tshark** — analyse fine des captures de paquets (PCAP), détection d'exfiltration de données
- **Suricata IDS/IPS** (intégré à OPNsense) — surveillance du trafic en temps réel, alertes par signatures
- **ModSecurity WAF** — protection et filtrage des requêtes applicatives web

### Gestion des logs & SIEM
- **Wazuh SIEM + Wazuh Dashboard** (Elasticsearch/OpenSearch) — colonne vertébrale du lab : centralisation des logs, File Integrity Monitoring (FIM), détection de rootkits et de menaces

### Réponse aux incidents & SOAR
- **TheHive / Shuffle SOAR** — création de cas d'investigation, gestion des tickets, suivi des étapes de remédiation

### Évaluation des vulnérabilités
- **Nessus / OpenVAS** — analyse des vulnérabilités de l'infrastructure et des serveurs web

### Analyse de malwares & forensique numérique
- **Any.Run / Cuckoo Sandbox** — analyse d'échantillons malveillants et de payloads, extraction d'IoCs
- **Volatility Framework** — analyse forensique de la mémoire vive (RAM), détection d'injection de processus

## 🔄 3. Scénarios Pratiques Complets (Attaque & Défense)

### 🎯 Scénario 1 — Attaques Réseau & Couche 2 (ARP Spoofing / MITM)

| Phase | Détails |
|---|---|
| **Attaque (Red Team)** | Empoisonnement de la table ARP via `Kali Linux` / `Bettercap` pour intercepter la communication entre la victime et la passerelle |
| **Détection (Blue Team)** | Surveillance des diffusions ARP suspectes via **Suricata IDS** et **OPNsense** ; capture et analyse des paquets avec **Wireshark** |
| **Défense & Réponse** | Activation du **Dynamic ARP Inspection (DAI)**, vérification des ports du switch, application de règles de blocage immédiates au niveau du pare-feu |

### 🎯 Scénario 2 — Attaques Applications Web (SQLi / RCE)

| Phase | Détails |
|---|---|
| **Attaque (Red Team)** | Exploitation de champs de saisie non sécurisés via `sqlmap` / `Burp Suite` / `Metasploit` pour exécuter des requêtes SQL malveillantes ou du RCE sur un serveur web en DMZ |
| **Détection (Blue Team)** | **Wazuh** capture les erreurs répétées et les patterns suspects dans les logs web (accès & erreurs) ; **Wazuh FIM** détecte toute modification non autorisée des fichiers système/application |
| **Défense & Réponse** | Activation du **ModSecurity WAF** pour bloquer automatiquement les requêtes malveillantes, application du principe de moindre privilège sur les comptes du serveur web |

### 🎯 Scénario 3 — Attaques Cloud & Conteneurs (Container Escape)

| Phase | Détails |
|---|---|
| **Attaque (Red Team)** | Tentative d'évasion de conteneurs **Docker** ou exploitation de mauvaises configurations IAM |
| **Détection (Blue Team)** | Surveillance du comportement des conteneurs via **Wazuh Docker Listener**, détection des tentatives d'élévation de privilèges vers l'hôte |
| **Défense & Réponse** | Exécution des conteneurs sans privilèges root, audit de configuration cloud avec des outils comme **Trivy** |

## 📋 4. Cycle d'Investigation Méthodique de l'Analyste SOC (Workflow TDIR)

Pour chaque alerte de sécurité déclenchée dans le laboratoire, l'analyste SOC suit un processus structuré en trois étapes :

1. **Détection**
   Repérage de l'anomalie via **Wazuh SIEM** ou **Suricata**.

2. **Triage & Analyse Préliminaire**
   - Ouverture du **Wazuh Dashboard** pour évaluer la sévérité de l'alerte
   - Étude du trafic via **Wireshark** et examen des logs
   - Analyse mémoire ou d'échantillons suspects via **Volatility** ou un sandbox

3. **Confinement & Remédiation**
   - Isolation immédiate de la machine compromise (isolation réseau) via des règles **OPNsense**
   - Correction de la vulnérabilité, clôture du dossier et documentation du ticket via **TheHive**

## 📂 Structure du dépôt (suggestion)

```
.
├── architecture/
│   ├── network-diagrams/
│   └── vlan-configuration/
├── scenarios/
│   ├── 01-arp-spoofing-mitm/
│   ├── 02-web-app-sqli-rce/
│   └── 03-cloud-container-escape/
├── tools/
│   ├── siem-wazuh/
│   ├── ids-suricata/
│   └── soar-thehive/
└── docs/
    └── writeups/
```

## 🎯 Contexte

Projet mené dans le cadre d'un parcours de spécialisation en **ingénierie de la sécurité et des opérations (SecOps / SOC Analyst / Blue & Red Team)**, avec pour objectif de reproduire les conditions opérationnelles d'un centre d'opérations de sécurité en entreprise.

## ⚠️ Disclaimer

Ce laboratoire est un environnement **strictement isolé et à usage éducatif**. Toutes les techniques présentées sont appliquées uniquement sur des systèmes appartenant à l'auteur, dans un cadre de recherche défensive et d'apprentissage.

---
---

# 🛡️ Projet 1 — Cybersecurity Lab Infrastructure & Dual-Perspective Portfolio (Red & Blue Teaming)

![Status](https://img.shields.io/badge/status-réalisé-success)
![Focus](https://img.shields.io/badge/focus-Red%20%2F%20Blue%20Team-critical)
![Stack](https://img.shields.io/badge/stack-Proxmox%20%7C%20OPNsense%20%7C%20Wazuh-informational)

## 📌 Introduction & Overview

Ce projet documente la conception, la construction et l'exploitation d'un laboratoire de cybersécurité interconnecté.

L'objectif du projet est d'adopter une **approche duale (Red Teaming vs Blue Teaming)** afin de comprendre le cycle de vie complet d'une cyberattaque :

1. **Simulation** — reconnaissance, exploitation, persistance
2. **Détection** — analyse et alerting via SIEM/IDS
3. **Réponse** — analyse forensique et mitigation

Chaque exercice du lab est pensé pour être reproductible et documenté selon une méthodologie stricte, dans une logique de formation continue en SecOps.

## 🏗️ Architecture du Laboratoire

Le laboratoire simule une infrastructure d'entreprise segmentée à l'aide de virtualisation avancée.

| Composant | Rôle | Outils |
|---|---|---|
| **Hyperviseur / Routeur** | Virtualisation & filtrage réseau | Proxmox VE, OPNsense (pare-feu, VLAN, IDS/IPS Suricata) |
| **Plateforme d'Attaque (Red Team)** | Simulation d'attaques | Kali Linux (VM), adaptateurs Wi-Fi en mode monitor |
| **Cibles & Services (Blue Team)** | Environnement à défendre | Alpine Linux, serveurs de test, conteneurs Docker, routeurs OpenWrt |
| **Supervision & SIEM** | Détection & analyse | Wazuh (HIDS/SIEM), logs de pare-feu centralisés |

## 🔴 Partie 1 — Offensive Security (Red Teaming & Simulations)

Tactiques, techniques et procédures (TTPs) simulées pour tester la résilience des systèmes.

### 1. Attaques Réseau & Couche 2 (Layer 2 & MitM)
- **ARP Spoofing / Poisoning** — interception de trafic via `Bettercap`, `Ettercap`
- **MAC Flooding & CAM Table Poisoning** — saturation des tables de commutation avec `macof`
- **Attaques Sans-Fil (Wi-Fi)** — Rogue AP / Evil Twin (`Airgeddon`, `Hostapd`), désauthentification (`aireplay-ng`), cassage WPA/WPA2
- **Bluetooth Exploitation** — scans et tests via `hcitool`, `sdptool`, attaques de type Bluesnarfing

### 2. Attaques Web & Applications
- **Injections & vulnérabilités** — XSS (Stored, Reflected, DOM-based), SQL Injection (`sqlmap`), Directory Traversal (`DotDotPwn`)
- **Authentification & session** — Session Hijacking, Bruteforcing, CSRF

### 3. Exploitation & Post-Exploitation
- **Buffer Overflow & RCE** — via `Metasploit` et `Meterpreter`
- **Privilege Escalation** — énumération via `LinPEAS`, `WinPEAS`
- **Malwares & Rootkits** — charges utiles (`msfvenom`), analyse de rootkits et malware fileless

## 🔵 Partie 2 — Defend & Engineering (Blue Teaming & Détection)

Pour chaque attaque simulée, une contre-mesure, une règle de détection et un durcissement (hardening) correspondants sont documentés.

### 1. Sécurité Réseau & Filtrage (OPNsense / Suricata)
- **DHCP Snooping & Dynamic ARP Inspection (DAI)** — atténuation de l'ARP Spoofing et des faux serveurs DHCP
- **IDS/IPS Suricata** — règles personnalisées pour détecter scans de ports (`nmap`), SYN Flood, anomalies de trafic

### 2. Surveillance des Hôtes & Réponse à Incident (Wazuh SIEM & HIDS)
- **File Integrity Monitoring (FIM)** — surveillance temps réel des fichiers sensibles (`/etc/passwd`, `/etc/shadow`)
- **Log Analysis** — détection de brute force SSH et d'élévations de privilèges
- **Audit & Hardening** — `Lynis` et playbooks `Ansible` automatisés

### 3. Analyse Forensique & Investigation
- **Analyse mémoire** — `Volatility` pour inspecter des images RAM et identifier des processus malveillants
- **Analyse de paquets** — `Wireshark` pour identifier des indicateurs de compromission (IoCs)

## 🚀 Méthodologie de Documentation

Chaque exercice réalisé dans ce lab suit une structure rigoureuse en trois temps :

1. **The Attack Scenario** — objectif de la simulation et outils utilisés
2. **The Detection Engineering** — comment le SIEM (Wazuh) ou l'IDS (Suricata) a capté l'événement
3. **The Incident Response & Mitigation** — mesures correctives, patch management, renforcement des règles

## 📂 Structure du dépôt (suggestion)

```
.
├── red-team/
│   ├── network-layer2/
│   ├── web-attacks/
│   └── post-exploitation/
├── blue-team/
│   ├── network-defense/
│   ├── host-monitoring/
│   └── forensics/
└── docs/
    └── writeups/
```

## 🎯 Contexte

Projet mené dans le cadre d'un parcours de spécialisation en **ingénierie de la sécurité et des opérations (SecOps / Blue & Red Team)**.

## ⚠️ Disclaimer

Ce laboratoire est un environnement **strictement isolé et à usage éducatif**. Toutes les techniques présentées sont appliquées uniquement sur des systèmes appartenant à l'auteur, dans un cadre de recherche défensive et d'apprentissage.
