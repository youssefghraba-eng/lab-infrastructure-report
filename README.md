
# 🛡️ Youssef Ghraba — Cybersecurity Portfolio

[![Status](https://img.shields.io/badge/Status-Active-success)]()
[![Focus](https://img.shields.io/badge/Focus-Blue%20Team-blue)]()
[![Stack](https://img.shields.io/badge/Stack-Wazuh%20%7C%20Suricata%20%7C%20OPNsense-orange)]()

> Analyste SOC en spécialisation **Blue Team** | **Detection Engineering** & **Incident Response (TDIR)**

---

## 👋 À propos

Ce dépôt regroupe l'ensemble de mes projets de laboratoire en cybersécurité, menés dans le cadre de ma spécialisation en **SecOps / SOC Analyst / Blue & Red Team**.

Chaque projet est documenté intégralement et conservé dans ce portfolio afin de tracer l'évolution de mon apprentissage et de mes compétences techniques.

---

## 📑 Sommaire des Projets

| # | Projet | Statut | Description |
|---|---|---|---|
| 1 | [Enterprise SOC Lab 2026](./docs/Enterprise%20SOC%20Lab%202026%20—%20Rapport%20d'Avancement%20·%20Partie%201.md) | 🟡 En cours | Laboratoire SOC professionnel (SIEM, SOAR, TDIR) |
| 2 | [Cybersecurity Lab Infrastructure](./docs/) | 🟢 Réalisé | Laboratoire Red & Blue Teaming fondateur |
| 3 | [Atomic Red Team — T1071.004](./docs/rapport-atomic-red-team-t1071-004.md) | 🟢 Réalisé | Détection DNS Tunneling (3 tests) |
| 4 | [Atomic Red Team — T1202](./docs/Atomic%20Red%20Team-technique%20T1202.md) | 🟢 Réalisé | Détection Indirect Command Execution |
| 5 | [Rapport Sécurité Kali](./docs/rapport_securite_kali.txt) | 🟢 Réalisé | Tests offensifs |
| 6 | [Rapports Lab SOC (Parties 1-4)](./docs/) | 🟢 Réalisé | Construction du lab Wazuh/OPNsense |

---

## 🏗️ Stack Technique

**SIEM / HIDS :** Wazuh, Elasticsearch, OpenSearch  
**IDS / IPS :** Suricata, ModSecurity WAF  
**SOAR :** TheHive, Shuffle  
**Réseau :** OPNsense, VLANs, DHCP Snooping, DAI  
**Forensique :** Volatility, Wireshark, Tshark  
**Systèmes :** Linux (Ubuntu, Alpine, Rocky, Kali), Windows Server  
**Virtualisation :** Proxmox VE, Hyper-V  
**Detection Engineering :** Règles Wazuh custom, Règles Suricata custom  
**Framework :** MITRE ATT&CK (T1071.004, T1202)

---

## 🎯 Scénarios Documentés

- **ARP Spoofing / MITM** : Attaque (Bettercap), Détection (Suricata), Réponse (DAI)
- **SQLi / RCE** : Attaque (sqlmap, Metasploit), Détection (Wazuh FIM), Réponse (ModSecurity)
- **Container Escape** : Attaque (Docker), Détection (Wazuh Docker Listener)
- **DNS Tunneling (T1071.004)** : 3 tests Atomic Red Team, 7 règles Wazuh créées
- **Indirect Command Execution (T1202)** : Détection via Sysmon EID 1

---

## 🔗 Me Contacter

- **LinkedIn :** [linkedin.com/in/youssef-g-74b6b4213](https://linkedin.com/in/youssef-g-74b6b4213)
- **TryHackMe :** [youssef.ghraba](https://tryhackme.com/p/youssef.ghraba)
- **CyberDefenders :** youssef.ghraba
- **Email :** youssef.ghraba@gmail.com

---

## ⚠️ Disclaimer

Ce laboratoire est un environnement **strictement isolé** et à usage **éducatif**. Toutes les techniques présentées sont appliquées uniquement sur des systèmes appartenant à l'auteur, dans un cadre de **recherche défensive** et d'**apprentissage**.

---

⭐ **N'hésitez pas à explorer mes rapports et à me contacter pour toute opportunité en SOC / Blue Team / Detection Engineering.**
