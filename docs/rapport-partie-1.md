# 📊 Enterprise SOC Lab 2026 — Rapport d'Avancement · Partie 1

![Status](https://img.shields.io/badge/status-phase%201%20%26%202%20termin%C3%A9es-success)
![Focus](https://img.shields.io/badge/focus-Infrastructure%20%7C%20SIEM-critical)
![Stack](https://img.shields.io/badge/stack-Hyper--V%20%7C%20OPNsense%20%7C%20Wazuh%20%7C%20Docker-informational)

> **Infrastructure, Segmentation Réseau & Intégration SIEM**
> Préparé par **Youssef Ghraba** — Casablanca, Maroc — Août 2026

---

## 📌 1. Introduction & Objectifs

Ce rapport présente le premier bilan d'avancement du projet **Enterprise SOC Lab 2026**, un laboratoire personnel d'opérations de sécurité conçu pour reproduire les conditions réelles d'un centre d'opérations de sécurité (SOC) en entreprise.

L'objectif du projet est double :
- Construire une **infrastructure réseau segmentée et réaliste** (zones LAN, DMZ, WLAN, WAN)
- Dérouler des **scénarios d'attaque et de défense complets**, en suivant le cycle de vie d'un incident de sécurité — détection, investigation, confinement et remédiation (méthodologie **TDIR**)

À ce stade du projet, **deux phases ont été menées à terme** : la mise en place de l'infrastructure réseau segmentée, et le déploiement d'un pot de miel (honeypot) intégré à la plateforme de supervision Wazuh SIEM. Ce document détaille les travaux réalisés, les configurations mises en œuvre et les premiers résultats observés.

---

## 🏗️ 2. Phase 1 — Infrastructure & Segmentation Réseau

Cette phase a consisté à établir la fondation technique du laboratoire : un environnement virtualisé segmenté, isolé et contrôlé par un pare-feu central, condition indispensable pour mener ensuite des scénarios d'attaque et de défense réalistes.

### 2.1 Virtualisation & Zonage Réseau

| Élément | Détails |
|---|---|
| **Environnement de virtualisation** | Mise en place de la plateforme **Hyper-V** pour la construction et la gestion de l'infrastructure virtuelle du laboratoire ainsi que la préparation des systèmes cibles |
| **Zone LAN (Trusted Zone)** | Configuration du réseau interne sécurisé hébergeant les postes de travail de l'équipe de défense (Blue Team) |
| **Zone DMZ (Demilitarized Zone)** | Isolement des serveurs et services exposés (ex. `Ubuntu_DMZ_Server`) dans une zone démilitarisée, protégeant ainsi le réseau interne de toute compromission directe |
| **Zones WLAN & WAN** | Configuration des interfaces réseau nécessaires à la simulation de l'accès Internet externe et aux tests de réseaux sans fil |

### 2.2 Déploiement & Configuration du Pare-feu OPNsense

- **Pare-feu central** — Déploiement d'**OPNsense** comme point central de routage, de surveillance et de filtrage du trafic entre l'ensemble des VLANs du laboratoire.
- **Gestion des alias** — Création d'objets de référence (ex. `Ubuntu_DMZ_IP`) afin de faciliter la gestion des adresses et l'écriture de règles de sécurité claires et maintenables.
- **Routage statique avancé** — Mise en place de routes statiques entre le système hôte (Windows) et les réseaux virtuels, garantissant une communication fluide et sécurisée vers la zone DMZ.

**Validation de la connectivité vers la zone DMZ (PowerShell) :**

```powershell
PS C:\Users\grimj> route -p add 192.168.100.0 mask 255.255.255.0 192.168.56.1
 OK!
PS C:\Users\grimj> Test-NetConnection -ComputerName 192.168.100.10 -Port 22

ComputerName     : 192.168.100.10
RemoteAddress    : 192.168.100.10
RemotePort       : 22
InterfaceAlias   : vEthernet (LAN_Net)
SourceAddress    : 192.168.56.2
TcpTestSucceeded : True
```

Le test ci-dessus confirme l'établissement réussi d'une route statique persistante vers la zone DMZ ainsi que l'accessibilité du service SSH (port 22) du serveur cible, validant la cohérence du plan d'adressage et des règles de routage.

### 2.3 Surveillance en Temps Réel du Trafic

Afin de valider le bon fonctionnement des règles de pare-feu et de vérifier la cohérence du trafic autorisé entre zones, une analyse en direct des journaux du pare-feu a été réalisée via l'interface OPNsense.

![Vue en direct des journaux du pare-feu OPNsense](images/01-opnsense-firewall-live-view.png)


<img width="1907" height="998" alt="Capture d&#39;écran 2026-08-29 005035" src="https://github.com/user-attachments/assets/8e788c9b-de46-4370-95fd-67e68bdd199a" />

<img width="1908" height="991" alt="Capture d&#39;écran 2026-08-29 001535" src="https://github.com/user-attachments/assets/c1e1d5a4-9c1c-47aa-97e8-7c9e0ec0cbc1" />




*Figure 1 — Vue en direct des journaux du pare-feu OPNsense : trafic autorisé entre la zone LAN et la zone DMZ.*

Cette capture illustre le flux de connexions TCP autorisées entre les interfaces LAN, WLAN et la zone DMZ, confirmant que la segmentation réseau et les règles de pare-feu fonctionnent conformément à la conception établie.

---

## 🍯 3. Phase 2 — Déploiement du Honeypot & Intégration SIEM

Cette deuxième phase avait pour objectif de déployer un leurre (honeypot) destiné à attirer et enregistrer les tentatives d'intrusion, puis de le connecter à la plateforme de supervision centralisée **Wazuh SIEM** afin de permettre une détection et une analyse en temps réel des événements de sécurité.

### 3.1 Déploiement du Honeypot SSH (Cowrie)

- **Moteur de conteneurisation** — Installation et configuration de `docker.io` et `docker-compose-v2` sur le serveur Ubuntu de la zone DMZ.
- **Configuration de l'environnement** — Création du répertoire de travail et configuration du fichier `docker-compose.yml` pour l'exécution du honeypot **Cowrie** sur le port simulé `2222`.
- **Vérification des attaques en direct** — Observation de tentatives d'intrusion simulées, incluant des authentifications échouées et réussies (`root/hunter`, `root/kkgkkgkg`), avec suivi détaillé des sessions interactives (journaux TTY).

**Extrait des journaux d'attaque simulée (Cowrie) :**

```
[ssh,dac948ed4e41,192.168.56.47] login attempt [root/hunter] succeeded
[HoneyPotSSHTransport,0,192.168.56.47] Initialized emulated server
                                        as architecture: linux-x64-lsb
```

### 3.2 Intégration & Supervision via Wazuh SIEM

Le serveur DMZ hébergeant le honeypot a été équipé d'un agent Wazuh, permettant une remontée centralisée des événements vers le tableau de bord Wazuh SIEM aux côtés des autres points de terminaison du laboratoire (pare-feu OPNsense inclus).

![Tableau de bord Wazuh - Agents actifs](images/02-wazuh-agents-dashboard.png)
*Figure 2 — Tableau de bord Wazuh : 3 agents actifs, incluant Ubuntu-DMZ-Server et OPNsense-firewall.*

Le tableau de bord confirme le statut actif de l'ensemble des agents déployés, avec une répartition claire des systèmes surveillés (Ubuntu, BSD/OPNsense) et une visibilité centralisée sur l'état de santé du parc.

![Journal des événements de sécurité Wazuh](images/03-wazuh-security-events.png)
*Figure 3 — Journal des événements de sécurité détectés sur Ubuntu-DMZ-Server (authentifications SSH, sessions PAM, anomalies de vérification racine).*

Sur une fenêtre de 24 heures, **29 événements** ont été capturés sur le serveur DMZ, incluant des réussites d'authentification SSH, des ouvertures/fermetures de session PAM et des alertes de détection d'anomalie basées sur l'hôte — validant la capacité du SIEM à collecter et corréler les événements liés aux tentatives d'intrusion sur le honeypot.

![Journaux bruts du honeypot Cowrie](images/04-cowrie-honeypot-logs.png)
*Figure 4 — Journaux bruts du honeypot Cowrie (terminal) : trace complète d'une session d'attaque simulée, de la connexion à la fermeture.*

Cette capture illustre le niveau de détail capturé par le honeypot : établissement de la session SSH, tentative d'authentification, ouverture d'un shell émulé, exécution de commandes et fermeture de la connexion — autant de données exploitables pour l'analyse comportementale des attaquants simulés.

---

## ✅ 4. Synthèse & Prochaines Étapes

À l'issue de ces deux premières phases, le laboratoire dispose désormais d'une infrastructure réseau segmentée et fonctionnelle, d'un pare-feu central opérationnel, ainsi que d'un dispositif de leurre (honeypot) pleinement intégré à une plateforme SIEM capable de détecter et de journaliser les tentatives d'intrusion en temps réel.

### Travaux réalisés

- [x] Architecture réseau segmentée (LAN / DMZ / WLAN / WAN) opérationnelle sous Hyper-V
- [x] Pare-feu OPNsense configuré, avec alias et routage statique validés
- [x] Honeypot SSH Cowrie déployé en conteneur Docker dans la zone DMZ
- [x] Intégration Wazuh SIEM opérationnelle sur l'ensemble des points de terminaison (3 agents actifs)
- [x] Première collecte et analyse d'événements de sécurité générés par des attaques simulées

### Prochaines étapes

- [ ] **Suricata IDS/IPS** — Déploiement de règles de détection personnalisées pour enrichir la corrélation d'événements réseau
- [ ] **Scénarios Red Team** — Exécution des scénarios d'attaque planifiés (ARP Spoofing, injections Web SQLi/RCE, évasion de conteneurs) avec documentation TDIR complète
- [ ] **SOAR & Réponse à incident** — Intégration de TheHive pour la gestion structurée des cas d'investigation et le suivi des remédiations

Ce premier rapport clôture la phase fondatrice du projet. Les prochaines livraisons documenteront l'exécution des scénarios offensifs et les réponses défensives associées, conformément à la méthodologie TDIR adoptée pour l'ensemble du laboratoire.

---

📎 [Retour au README principal du projet](../README.md) · 🔗 [github.com/youssefghraba-eng](https://github.com/youssefghraba-eng)
