— Cybersecurity Lab Infrastructure & Dual-Perspective Portfolio (Red & Blue Teaming)

![Status](https://img.shields.io/badge/status-active-success)
![Focus](https://img.shields.io/badge/focus-Red%20%2F%20Blue%20Team-critical)
![Stack](https://img.shields.io/badge/stack-Proxmox%20%7C%20OPNsense%20%7C%20Wazuh-informational)

## 📌 Introduction & Overview

Ce dépôt documente la conception, la construction et l'exploitation d'un **laboratoire de cybersécurité interconnecté**, nommé **المدرعة النووية**.

L'objectif du projet est d'adopter une **approche duale (Red Teaming vs Blue Teaming)** afin de comprendre le cycle de vie complet d'une cyberattaque :

1. **Simulation** — reconnaissance, exploitation, persistance
2. **Détection** — analyse et alerting via SIEM/IDS
3. **Réponse** — analyse forensique et mitigation

Chaque exercice du lab est pensé pour être reproductible et documenté selon une méthodologie stricte, dans une logique de formation continue en SecOps.

---

## 🏗️ Architecture du Laboratoire

Le laboratoire simule une infrastructure d'entreprise segmentée à l'aide de virtualisation avancée.

| Composant | Rôle | Outils |
|---|---|---|
| **Hyperviseur / Routeur** | Virtualisation & filtrage réseau | Proxmox VE, OPNsense (pare-feu, VLAN, IDS/IPS Suricata) |
| **Plateforme d'Attaque (Red Team)** | Simulation d'attaques | Kali Linux (VM), adaptateurs Wi-Fi en mode monitor |
| **Cibles & Services (Blue Team)** | Environnement à défendre | Alpine Linux, serveurs de test, conteneurs Docker, routeurs OpenWrt |
| **Supervision & SIEM** | Détection & analyse | Wazuh (HIDS/SIEM), logs de pare-feu centralisés |

---

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

---

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

---

## 🚀 Méthodologie de Documentation

Chaque exercice réalisé dans ce lab suit une structure rigoureuse en trois temps :

1. **The Attack Scenario** — objectif de la simulation et outils utilisés
2. **The Detection Engineering** — comment le SIEM (Wazuh) ou l'IDS (Suricata) a capté l'événement
3. **The Incident Response & Mitigation** — mesures correctives, patch management, renforcement des règles

---

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

---

## 🎯 Contexte

Projet mené dans le cadre d'un parcours de spécialisation en **ingénierie de la sécurité et des opérations (SecOps / Blue & Red Team)**.

---

## ⚠️ Disclaimer

Ce laboratoire est un environnement **strictement isolé et à usage éducatif**. Toutes les techniques présentées sont appliquées uniquement sur des systèmes appartenant à l'auteur, dans un cadre de recherche défensive et d'apprentissage.
