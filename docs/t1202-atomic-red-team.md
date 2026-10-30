# 📊 Enterprise SOC Lab 2026 — Atomic Red Team----technique T1202

![Status](https://img.shields.io/badge/status-simulation%20T1202%20termin%C3%A9e-success)
![Focus](https://img.shields.io/badge/focus-Atomic%20Red%20Team%20%7C%20TDIR-critical)
![Stack](https://img.shields.io/badge/stack-Windows%20Server%20Core%20%7C%20Sysmon%20%7C%20Wazuh%20%7C%20Suricata-informational)

> **Simulation ATT&CK T1202 (Indirect Command Execution) sur Windows Server Core**
> Préparé par **Youssef Ghraba** — Septembre 2026

---

## 📌 1. Introduction & Objectifs

Dans le cadre du développement de mes compétences en ingénierie de détection et d'investigation d'incidents (**TDIR**), j'ai mis en place et testé un environnement de laboratoire personnel, reposant sur **Windows Server Core**. L'objectif de ce travail est de simuler des techniques d'attaque réelles documentées dans le référentiel **Atomic Red Team**, en ciblant spécifiquement la technique **T1202 — Indirect Command Execution**.

Cette expérimentation vise à :
- Analyser le comportement des outils système natifs de Windows détournés à des fins offensives ;
- Vérifier l'efficacité des couches de détection et d'analyse via la collecte de télémétrie avec **Sysmon** ;
- Centraliser et corréler les journaux via **Wazuh SIEM** ;
- Surveiller le trafic réseau associé via **Suricata** ;
- Documenter dans quelle mesure les contraintes structurelles d'un environnement **Server Core** réduisent la surface d'attaque ;
- Construire et ajuster des règles de détection (Rule Tuning) adaptées aux menaces observées.

| Élément | Détails |
|---|---|
| **Technique MITRE ATT&CK** | T1202 — Indirect Command Execution |
| **Environnement cible** | `server01-core` — Windows Server Core |
| **Télémétrie** | Sysmon (Event ID 1 — Process Creation) |
| **SIEM** | Wazuh (agent 008 — `192.168.56.58`) |
| **IDS réseau** | Suricata |

---

## 🧪 2. Scénarios d'Exécution Indirecte de Commandes (T1202)

Quatre techniques natives de Windows ont été testées afin d'évaluer leur exploitabilité et leur visibilité dans l'environnement Server Core.

### 2.1 Test 1 — `pcalua.exe` (Program Compatibility Assistant)

| Élément | Détails |
|---|---|
| **Outil ciblé** | Assistant de compatibilité des programmes Windows |
| **Fonctionnement** | Capable de lancer des programmes et des commandes depuis la ligne de commande |
| **Commande testée** | `pcalua.exe -a calc.exe` |
| **Objectif défensif** | Surveiller toute utilisation suspecte de `pcalua.exe` pour lancer des programmes inhabituels |

**Vérification préalable de la présence du binaire :**

```powershell
PS C:\Users\Administrator> Test-Path "C:\Windows\System32\pcalua.exe"
False
```

**Résultat :** `pcalua.exe` est **absent** de Server Core. Windows Server Core étant une édition allégée et dépourvue des interfaces et outils non essentiels (comme `pcalua.exe` ou la calculatrice traditionnelle), cette absence constitue en soi une excellente illustration pratique du **durcissement natif** des environnements d'entreprise.

---

### 2.2 Test 2 — `forfiles.exe` (succès de l'exécution indirecte)

`forfiles.exe` est un outil système natif dédié à la recherche et à la gestion de fichiers. Il est détourné par les attaquants comme méthode d'exécution indirecte de commandes : l'outil recherche un fichier donné (ex. `notepad.exe`) et exécute une commande alternative pour chaque résultat trouvé.

**Commande d'exécution (adaptée à Server Core, sans dépendance à la calculatrice) :**

```powershell
PS C:\Users\Administrator> forfiles /p c:\windows\system32 /m notepad.exe /c "cmd.exe /c echo T1202-Test-Success"

T1202-Test-Success
```

**Analyse tactique :**
- **Outil utilisé comme proxy** — `forfiles.exe` a été chargé de rechercher le fichier `notepad.exe` dans le dossier système.
- **Exécution indirecte** — pour chaque correspondance trouvée, l'outil n'a pas ouvert le Bloc-notes, mais a exécuté la commande spécifiée (`cmd.exe /c echo T1202-Test-Success`).
- **Résultat** — le texte s'est affiché à l'écran, démontrant comment un attaquant peut exploiter un outil système légitime pour exécuter des commandes furtives sans appel direct et conventionnel à `cmd.exe`.

#### 🔍 Détection via Wazuh SIEM

<img width="1906" height="865" alt="Capture d&#39;écran 2026-09-25 201238" src="https://github.com/user-attachments/assets/0c720876-1250-41a4-8b06-faeeb9694995" />

<img width="1912" height="1002" alt="Capture d&#39;écran 2026-09-25 202910" src="https://github.com/user-attachments/assets/bbbc1649-7d88-45a3-b56d-996c7b7186d0" />


Wazuh a capturé la simulation avec une précision totale.

| Métrique | Valeur |
|---|---|
| **Total d'événements enregistrés** (agent `server01-core`) | 937 |
| **Règle déclenchée** | `92052` — *Invite de commande Windows lancée par un processus anormal* |
| **Règle complémentaire** | `92032` — *Exécution suspecte du shell cmd Windows* |
| **Technique MITRE associée** | T1059.003 (Windows Command Shell) / T1202 |

**Extrait détaillé de l'alerte Sysmon (Event ID 1 — Process Creation) :**

| Champ | Valeur |
|---|---|
| `agent.name` | `server01-core` |
| `agent.ip` | `192.168.56.58` |
| `data.win.eventdata.image` | `C:\Windows\System32\cmd.exe` |
| `data.win.eventdata.commandLine` | `/c echo T1202-Test-Success` |
| `data.win.eventdata.parentImage` | `C:\Windows\System32\forfiles.exe` |
| `data.win.eventdata.parentCommandLine` | `"C:\Windows\system32\forfiles.exe" /p c:\windows\system32 /m notepad.exe /c "cmd.exe /c echo T1202-Test-Success"` |
| `data.win.eventdata.parentUser` | `SERVER01-CORE\Administrateur` |
| `data.win.eventdata.integrityLevel` | Haut |
| `rule.id` | `92052` |
| `rule.level` | 4 |
| `rule.mitre.id` | T1059.003 |
| `rule.mitre.tactic` | Exécution |
| `rule.mitre.technique` | Shell de commandes Windows |
| `rule.firedtimes` | 4 |

Cette corrélation confirme que Wazuh a correctement identifié le lien de filiation anormal entre `forfiles.exe` (processus parent) et `cmd.exe` (processus enfant) — signature caractéristique d'une exécution indirecte de commande.

#### 🛡️ Cycle de Réponse à Incident (TDIR)

**1. Confinement (Containment)**

En environnement de production, l'isolement se ferait en un clic depuis Wazuh. Sur le laboratoire, cela a été simulé via :
- Arrêt du service agent Wazuh ou désactivation de l'interface réseau virtuelle du serveur ciblé ;
- Terminaison manuelle du processus anormal :

```dos
taskkill /PID 3912 /F
```

**2. Éradication & Remédiation**

- Vérification de l'absence de fichiers temporaires ou de résidus liés à l'exécution de `forfiles.exe` dans `C:\Windows\System32\`.

**3. Ajustement des règles de détection (Rule Tuning)**

Étape clé de l'ingénierie TDIR : création d'une règle personnalisée Wazuh pour détecter toute tentative similaire d'évasion via d'autres outils (`forfiles.exe`, `pcalua.exe`, etc.).

```bash
cd /var/ossec/etc/rules/
# création ou édition de local_rules.xml
```

```xml
<group name="windows, sysmon, attack_evasion,">
  <rule id="100101" level="10">
    <if_sid>92052</if_sid>
    <field name="win.eventdata.parentImage">.*\\forfiles\.exe</field>
    <description>Atomic Red Team Simulation: Indirect Command Execution via forfiles.exe detected on server01-core.</description>
    <mitre>
      <id>T1202</id>
    </mitre>
  </rule>
</group>
```

```bash
systemctl restart wazuh-manager
```

Cette démarche complète le cycle de vie d'un incident de sécurité dans son intégralité : **Détection → Triage → Confinement → Ajustement des règles**, et pas seulement une observation passive des alertes.

---

### 2.3 Test 3 — `conhost.exe` (détection manquée — angle mort identifié)

`conhost.exe` (Console Window Host) est un processus légitime et fondamental de Windows, chargé de gérer les fenêtres graphiques de `cmd.exe` ou `powershell.exe`. En fonctionnement normal, ce processus **ne devrait jamais** créer ou lancer d'autres programmes en tant que processus parent — un tel comportement (anomalie de filiation de processus) est un indicateur classique d'évasion utilisé par les attaquants.

**Exécution :**

```powershell
PS C:\Users\Administrator> conhost.exe "notepad.exe"
```

**Résultat de détection :** ❌ **Aucune alerte générée dans Wazuh** pour ce test.

**Conclusion :** Server Core étant totalement dépourvu de gestionnaire de fenêtres graphiques et de services de bureau visuel, les composants responsables de l'exécution autonome de `conhost.exe` sont soit désactivés, soit refusés silencieusement par le système. La tentative d'exécution manuelle échoue donc **silencieusement**, sans transiter par les canaux surveillés par les solutions EDR/Sysmon — révélant un angle mort de détection propre à cet environnement.

---

### 2.4 Test 4 — Registre RunMRU (absence d'artefacts)

Lorsqu'un utilisateur saisit une commande dans la fenêtre **« Exécuter »** (Run), Windows enregistre immédiatement cette commande dans la clé de registre :

```
HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\RunMRU
```

Les attaquants exploitent parfois cette méthode (ou la simulent) pour masquer leurs actions, tandis que les analystes recherchent les clés **MRU** (Most Recently Used) car elles constituent des preuves médico-légales (Forensic Artifacts) des commandes récemment exécutées.

**Vérification de la clé de registre :**

```powershell
PS C:\Users\Administrator> Get-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\RunMRU" -ErrorAction SilentlyContinue
```

**Résultat :** aucune sortie — la clé est totalement absente.

**Analyse technique :** ce résultat est logique à 100 % dans un environnement Server Core, pour deux raisons :

1. **Absence de shell Explorer** — la clé `RunMRU` est intrinsèquement liée à `explorer.exe`. Le processus Explorer de base étant absent sur un serveur Headless, la fenêtre « Exécuter » traditionnelle n'a jamais été sollicitée, donc la clé n'a jamais été créée.
2. **Absence d'artefacts** — l'absence d'interaction humaine via une interface graphique signifie qu'aucun historique de commandes récentes n'a été journalisé.

Ce résultat démontre une nouvelle fois la robustesse architecturale de Server Core : la surface d'attaque associée à `RunMRU` et à l'historisation des commandes visuelles est **totalement supprimée à la racine**, comparée aux versions Windows classiques (Client/Desktop).

---

## ✅ 3. Synthèse & Enseignements

| Test | Outil | Résultat | Détection Wazuh |
|---|---|---|---|
| 1 | `pcalua.exe` | Outil absent (échec d'exécution) | N/A — surface d'attaque supprimée |
| 2 | `forfiles.exe` | Exécution indirecte réussie | ✅ Détecté (règles 92052 / 92032) |
| 3 | `conhost.exe` | Exécution manuelle silencieuse | ❌ Non détecté — angle mort identifié |
| 4 | `RunMRU` (Registry) | Aucun artefact généré | N/A — surface d'attaque supprimée |

**Enseignements clés :**
- Les contraintes structurelles de **Windows Server Core** réduisent significativement la surface d'attaque disponible pour les techniques T1202 reposant sur des composants graphiques (`pcalua.exe`, `conhost.exe`, `RunMRU`).
- La technique **`forfiles.exe`** reste pleinement fonctionnelle et exploitable même sur Server Core, et a été détectée avec succès grâce à la télémétrie Sysmon corrélée dans Wazuh.
- Le test `conhost.exe` a révélé un **angle mort de détection** : une commande exécutée manuellement peut échouer silencieusement sans générer de télémétrie, ce qui constitue un point d'attention pour l'ingénierie de détection.
- La capacité à **construire une règle de détection personnalisée** (Rule Tuning) à partir d'une alerte existante est la compétence la plus critique de ce cycle d'exercice pour un ingénieur TDIR.

---


📎 [Retour au README principal du projet](../README.md) · 🔗 [github.com/youssefghraba-eng](https://github.com/youssefghraba-eng)
