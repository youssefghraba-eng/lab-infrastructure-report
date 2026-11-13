[README_Fraud_AML_KYC.md](https://github.com/user-attachments/files/34820620/README_Fraud_AML_KYC.md)
# 🛡️ Youssef Ghraba — Cyber Financial Crime Investigator | Fraud Monitoring & AML / KYC

![Focus](https://img.shields.io/badge/Focus-Fraud%20Monitoring%20%26%20AML%2FKYC-1B2A41?style=flat-square)
![Cyber](https://img.shields.io/badge/Cybers%C3%A9curit%C3%A9-D%C3%A9tection%20%26%20Investigation-C4954A?style=flat-square)
![Stack](https://img.shields.io/badge/Stack-Wazuh%20%7C%20Suricata%20%7C%20OPNsense-1F4E8C?style=flat-square)

> **17 ans en environnement financier**, dont près de **5 ans de supervision d'alertes transactionnelles en temps réel** (Fraud Supervisor), complétés par une pratique de la **détection et de l'investigation en cybersécurité**.

---

## 👋 À propos

Ce dépôt documente mes laboratoires de cybersécurité (SIEM, IDS/IPS, ingénierie de détection, investigation) et **leur lien avec mon métier : la surveillance de transactions, l'analyse de risques et la lutte contre la fraude**.

Les outils sont ceux d'un SOC ; la méthode est celle d'un analyste fraude :

**signal → alerte → triage → investigation → décision documentée**

Chaque projet est documenté de façon reproductible, afin de tracer l'évolution de mes compétences. Mon objectif : démontrer une maîtrise pratique de la détection, de l'analyse et de la réponse aux incidents, en environnement on-premise (extension cloud en préparation).

Extension prévue : simulation de détection de fraudes transactionnelles (Structuring) via un moteur de règles open-source (ezrules).

---

## 🔗 Du SOC à la lutte contre la fraude : cartographie

Chaque travail de la colonne de gauche a été réalisé dans mes laboratoires. La colonne centrale indique son équivalent dans la surveillance de fraude et l'AML.

| Travail réalisé (lab) | Équivalent Fraude / AML / KYC | Compétence démontrée |
|---|---|---|
| **7 règles Wazuh personnalisées** (DNS Tunneling, T1071.004) | Règles de filtrage et de scoring qui détectent un comportement anormal | Conception et ajustement de règles de détection |
| **Triage des alertes par priorité (P1 / P2)** selon le workflow TDIR | Qualification et priorisation des alertes transactionnelles par niveau de risque | Triage, priorisation, escalade |
| **Investigation** : logs, captures réseau (Wireshark), Sysmon, FIM Wazuh | Investigation d'un cas suspect : corrélation d'événements et reconstitution d'une chronologie | Analyse, corrélation, traçabilité (IOCs, timeline) |
| **Gestion des incidents par tickets** (GitHub Issues) et rapports d'incident | Gestion de cas et constitution de dossiers d'investigation | Documentation, rédaction de rapports, piste d'audit |
| **Playbooks / SOP** de réponse | Procédures opérationnelles standard de traitement des alertes | Rigueur procédurale, conformité aux procédures |
| **File Integrity Monitoring** (Wazuh FIM) | Détection de modifications non autorisées, contrôle d'intégrité | Contrôle et audit |
| **Scénarios d'attaque simulés** (ARP Spoofing, SQLi / RCE, DNS Tunneling…) | Compréhension des vecteurs techniques qui précèdent une fraude numérique (vol d'identifiants, accès non autorisé, exfiltration) | Anticipation des schémas de fraude « cyber » |
| **Rapports de construction du lab** (parties 1 à 4) | Documentation et traçabilité des décisions | Communication écrite, rigueur documentaire |

### 🧠 Pourquoi cette double approche ?

- Une fraude numérique commence souvent par un incident technique (phishing, vol d'identifiants, accès non autorisé) **avant** la transaction frauduleuse.
- Détecter tôt l'anomalie technique permet d'anticiper la fraude financière.
- Le raisonnement reste le même : repérer un écart, le qualifier, l'investiguer, le documenter.

---

## 🛡️ Ingénierie de détection : règles Wazuh (DNS Tunneling)

J'ai développé et testé des règles personnalisées Wazuh pour détecter les comportements suspects liés au DNS (MITRE T1071.004), avec des tests Atomic Red Team puis une session C2 dnscat2.

<!-- ⬇️ Conservez ici votre image « Guide des règles Wazuh » et votre lien « 👉 Voir le code XML des règles » actuels -->

---

## 📑 Sommaire des projets

| # | Projet | Statut | Description | Lien Fraude / AML |
|---|---|---|---|---|
| 1 | **Enterprise SOC Lab 2026** | 🟡 En cours | Laboratoire de détection et d'investigation (SIEM, IDS/IPS, TDIR) | Surveillance, triage et investigation d'alertes |
| 2 | **Infrastructure & Automatisation** | 🟢 Réalisé | Laboratoire Red & Blue Team, fondation du portfolio | Compréhension des techniques d'attaque pour mieux détecter |
| 3 | **Atomic Red Team : T1071.004 (Test #4)** | 🟢 Réalisé | Détection de DNS Tunneling (session C2 dnscat2) | Détection d'une communication / exfiltration discrète |
| 4 | **Atomic Red Team : T1202** | 🟢 Réalisé | Détection d'exécution indirecte de commande (Sysmon) | Détection d'un contournement des contrôles |
| 5 | **Rapport Sécurité Kali** | 🟢 Réalisé | Tests offensifs et analyse | Évaluation des risques techniques |
| 6 | **SOC Lab Wazuh / OPNsense** | 🟢 Réalisé | Construction du lab (parties 1 à 4) | Documentation et traçabilité |

---

## 🎯 Scénarios documentés

| Scénario | Attaque (simulée) | Détection | Réponse | Lecture Fraude / Risque |
|---|---|---|---|---|
| **ARP Spoofing / MITM** | Bettercap | Suricata | Dynamic ARP Inspection | Interception de sessions et d'identifiants (prise de contrôle de compte) |
| **SQLi / RCE** | sqlmap, Metasploit | Wazuh FIM + logs web | ModSecurity WAF | Accès non autorisé à des données clients ou transactionnelles |
| **Container Escape** | Docker | Wazuh Docker Listener | Exécution sans privilèges root | Compromission de l'infrastructure qui héberge les services |
| **DNS Tunneling (T1071.004)** | Atomic Red Team (3 tests) + session C2 dnscat2 | 7 règles Wazuh créées | : | Exfiltration discrète de données ou canal de commande |
| **Exécution indirecte (T1202)** | Atomic Red Team | Sysmon Event ID 1 | : | Contournement des contrôles de sécurité |

---

## 🏗️ Stack technique

| Catégorie | Outils |
|---|---|
| **SIEM / HIDS** | Wazuh, Elasticsearch, OpenSearch |
| **IDS / IPS / WAF** | Suricata, ModSecurity |
| **Gestion des cas** | GitHub Issues (investigations et playbooks) |
| **Réseau** | OPNsense, VLANs, DHCP Snooping, DAI |
| **Forensique** | Volatility, Wireshark, Tshark |
| **Systèmes** | Linux (Ubuntu, Alpine, Rocky, Kali), Windows Server |
| **Virtualisation** | Proxmox VE, Hyper-V |
| **Détection** | Règles Wazuh personnalisées, règles Suricata personnalisées |
| **Framework** | MITRE ATT&CK (T1071.004, T1202) |

---

## 🧭 Roadmap : prochains projets

### 🔵 Fraude & AML (prévu)
- [ ] **AML Transaction Monitoring Lab** : détection de schémas de type *structuring* avec un moteur de règles open-source, et documentation des cas d'investigation

### 🟢 Détection cyber (en cours)
- [ ] **Phishing Simulation (T1566)** : e-mail malveillant → exécution de payload (vecteur fréquent de fraude numérique)
- [ ] **Kerberoasting (T1558.003)** : extraction de tickets Kerberos (Wazuh, Sysmon)
- [ ] **Pass-the-Hash (T1550.002)** : mouvement latéral via hash NTLM (Wazuh)
- [ ] **C2 via HTTPS (T1071.001)** : canal C2 chiffré (Wazuh, Suricata)

### 🟡 Cloud Security (en préparation)
- [ ] Détection d'anomalies IAM (AWS CloudTrail → Wazuh)
- [ ] Mauvaises configurations S3 (AWS Config, Prowler)
- [ ] Audit logs Kubernetes

### 🔵 Detection Engineering
- [ ] 10 règles Wazuh personnalisées pour techniques MITRE
- [ ] 5 règles Suricata personnalisées pour C2 réseau
- [ ] Threat Hunting (Wazuh, ELK)
- [ ] SOAR : playbooks (Shuffle, TheHive)

> Les cases se cochent au fur et à mesure que chaque rapport est publié dans ce dépôt.

---

## 💼 Contexte professionnel

- **Fraud Supervisor, IPRC (groupe HPS)** : depuis nov. 2021. Surveillance en temps réel des alertes transactionnelles via le système PowerCard, qualification, escalade, optimisation des règles de filtrage et de scoring, encadrement d'équipe.
- **Conseiller de Clientèle, EQDOM** : janv. 2009 – nov. 2021. Gestion d'un portefeuille clients, analyse de dossiers de risque, traitement des litiges.

---

## 🔗 Me contacter

- **LinkedIn :** [linkedin.com/in/youssef-g-74b6b4213](https://www.linkedin.com/in/youssef-g-74b6b4213)
- **TryHackMe :** youssef.ghraba
- **CyberDefenders :** youssef.ghraba
- **Email :** youssef.ghraba@gmail.com

---

## ⚠️ Avertissement

Ces laboratoires sont des environnements strictement isolés, à usage éducatif. Toutes les techniques sont appliquées uniquement sur des systèmes appartenant à l'auteur, dans un cadre de recherche défensive et d'apprentissage. **Aucune donnée réelle de clients ou de transactions n'est utilisée ni publiée dans ce dépôt.**

⭐ N'hésitez pas à explorer mes rapports et à me contacter pour toute opportunité en **Fraud Monitoring, Analyse de risques, AML / KYC ou Cybersécurité**.
