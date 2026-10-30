# 🛡️ Youssef Ghraba — Portefeuille de cybersécurité

[![Status](https://img.shields.io/badge/Status-Actif-brightgreen)]()
[![Focus](https://img.shields.io/badge/Focus-Blue%20Team-blue)]()
[![Stack](https://img.shields.io/badge/Stack-Wazuh%20%7C%20Suricata%20%7C%20OPNsense-orange)]()

> **Analyste SOC en spécialisation Blue Team | Ingénierie de la détection et réponse aux incidents (TDIR)**

---

## 👋 À propos

Ce dépôt regroupe l'ensemble de mes projets de laboratoire en cybersécurité, menés dans le cadre de ma spécialisation en **SecOps / SOC Analyst / Blue & Red Team**.

Chaque projet est documenté intégralement et conservé dans ce portfolio afin de tracer l'évolution de mon apprentissage et de mes compétences techniques. Mon objectif est de démontrer une maîtrise pratique de la détection, de l'analyse et de la réponse aux incidents, en environnement on-premise et cloud.

---

## 📑 Sommaire des Projets

| # | Projet | Statut | Description |
|---|---|---|---|
| 1 | [Laboratoire SOC d'entreprise 2026](https://github.com/youssefghraba-eng/lab-infrastructure-report/blob/main/Docs/enterprise-soc-lab-partie1.md) | 🟡 En cours | Laboratoire SOC professionnel (SIEM, SOAR, TDIR) |
| 2 | [Infrastructure & Automatisation](https://github.com/youssefghraba-eng/lab-infrastructure-report/blob/main/Docs/infrastructure-automation.md) | 🟢 Réalisé | Laboratoire Rouge & Bleu, fondation de l'équipe |
| 3 | [Atomic Red — T1071.004 (Test #4)](https://github.com/youssefghraba-eng/lab-infrastructure-report/blob/main/Docs/rapport-atomic-red-team-t1071-004-test4.md) | 🟢 Réalisé | Détection DNS Tunneling (session C2 dnscat2) |
| 4 | [Atomic Red — T1202](https://github.com/youssefghraba-eng/lab-infrastructure-report/blob/main/Docs/t1202-atomic-red-team.md) | 🟢 Réalisé | Exécution indirecte de commande (détection) |
| 5 | [Rapport Sécurité Kali](https://github.com/youssefghraba-eng/lab-infrastructure-report/blob/main/Docs/rapport_securite_kali.txt) | 🟢 Réalisé | Tests offensifs et analyse |
| 6 | [SOC Lab Wazuh / OPNsense](https://github.com/youssefghraba-eng/lab-infrastructure-report/blob/main/Docs/rapport-lab-soc-wazuh-opnsense.md) | 🟢 Réalisé | Construction du lab (Parties 1-4) |

---

## 🏗️ Stack Technique

| Catégorie | Outils |
|---|---|
| **SIEM / HIDS** | Wazuh, Elasticsearch, OpenSearch |
| **IDS / IPS** | Suricata, ModSecurity WAF |
| **SOAR** | TheHive, Shuffle |
| **Réseau** | OPNsense, VLANs, DHCP Snooping, DAI |
| **Forensique** | Volatility, Wireshark, Tshark |
| **Systèmes** | Linux (Ubuntu, Alpine, Rocky, Kali), Windows Server |
| **Virtualisation** | Proxmox VE, Hyper-V |
| **Détection** | Règles Wazuh custom, Règles Suricata custom |
| **Framework** | MITRE ATT&CK (T1071.004, T1202) |

---

## 🎯 Scénarios Documentés

- **ARP Spoofing / MITM** : Attaque (Bettercap), Détection (Suricata), Réponse (DAI)
- **SQLi / RCE** : Attaque (sqlmap, Metasploit), Détection (Wazuh FIM), Réponse (ModSecurity)
- **Container Escape** : Attaque (Docker), Détection (Wazuh Docker Listener)
- **DNS Tunneling (T1071.004)** : 3 tests Atomic Red Team, 7 règles Wazuh créées
- **Exécution indirecte de commande (T1202)** : Détection via Sysmon EID 1

---

## ☁️ Cloud Security (en préparation)

La sécurité cloud est une extension naturelle de mon parcours SOC. Les rapports détaillés et les démonstrations pratiques (attaque → détection → analyse) seront progressivement publiés dans ce dépôt. Chaque scénario suivra la même méthodologie rigoureuse que les projets on-premise, avec documentation complète, captures d'écran et règles de détection associées.

Les scénarios prévus incluent :
- **Détection des anomalies IAM (AWS)** : Surveillance des créations d'utilisateurs, escalades de privilèges et accès suspects via AWS CloudTrail, avec ingestion dans Wazuh.
- **Mauvaise configuration S3** : Détection de buckets S3 publics ou mal configurés (AWS Config, Prowler) et réponse automatisée.
- **Sécurité des conteneurs (Docker)** : Détection d'évasion de conteneur et de comportements suspects via Wazuh Docker Listener.
- **Audit Logs Kubernetes** : Détection d'activités suspectes dans un cluster K8s (accès non autorisé, exécution de commandes).

---

## 🚀 Roadmap — Prochains projets

### 🟢 Niveau 1 — SOC Analyst (en cours)

- [ ] **Kerberoasting (T1558.003)** : Extraction de tickets Kerberos (Wazuh, Sysmon, Rubeus)
- [ ] **Pass-the-Hash (T1550.002)** : Mouvement latéral via hash NTLM (Wazuh, Mimikatz)
- [ ] **Phishing Simulation (T1566)** : Email malveillant → exécution de payload
- [ ] **C2 via HTTPS (T1071.001)** : Canal C2 chiffré (Wazuh, Suricata, Sliver)

### 🟡 Niveau 2 — Cloud Security (en préparation)

### 🔵 Niveau 3 — Detection Engineering

- [ ] **10 règles Wazuh custom** pour techniques MITRE
- [ ] **5 règles Suricata custom** pour C2 réseau
- [ ] **Threat Hunting** (Wazuh, ELK)
- [ ] **SOAR Playbook** (Shuffle, TheHive)

---

## 🔗 Me Contacter

- **LinkedIn** : [linkedin.com/in/youssef-g-74b6b4213](https://linkedin.com/in/youssef-g-74b6b4213)
- **TryHackMe** : [youssef.ghraba](https://tryhackme.com/p/youssef.ghraba)
- **CyberDefenders** : [youssef.ghraba](https://cyberdefenders.org/profile/youssef.ghraba)
- **Email** : youssef.ghraba@gmail.com

---

## ⚠️ Avertissement

Ce laboratoire est un environnement **strictement isolé et à usage éducatif**. Toutes les techniques présentées sont appliquées uniquement sur des systèmes appartenant à l'auteur, dans un cadre de recherche défensive et d'apprentissage.

---

⭐ **N'hésitez pas à explorer mes rapports et à me contacter pour toute opportunité en SOC / Blue Team / Detection Engineering.**
