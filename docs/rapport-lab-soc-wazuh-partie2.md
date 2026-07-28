# Rapport de Projet — Home Lab SOC (Wazuh + Ansible + Tailscale) — Partie 2

**Auteur :** Grim Jow
**Date :** 28 juillet 2026
**Suite de :** *Rapport 1 — Déploiement Wazuh + OPNsense + Ansible*

---

## 1. Objectif de cette session

La partie 1 avait permis d'installer Wazuh Manager (Rocky Linux) et les agents sur Ubuntu Server, Parrot OS et Kali Linux via Ansible. Cette session s'est concentrée sur trois points :

1. Diagnostiquer et corriger l'échec d'enrôlement des agents (aucun agent visible côté Manager).
2. Aligner les versions Agent/Manager pour rétablir une communication stable.
3. Préparer un scénario d'attaque en reliant Kali Linux (VMware) à Ubuntu Server (Hyper-V) via un tunnel réseau, malgré leur isolation dans deux hyperviseurs différents.

---

## 2. Diagnostic initial : agents installés mais invisibles côté Manager

Après le premier playbook Ansible complet (installation + activation du service), la vérification côté Manager ne montrait aucun agent :

```bash
sudo /var/ossec/bin/agent_control -l
```

```
Wazuh agent_control. List of available agents:
   ID: 000, Name: localhost.localdomain (server), IP: 127.0.0.1, Active/Local
```

Pourtant, côté Parrot OS et Ubuntu Server, `systemctl status wazuh-agent` indiquait bien un service **actif (running)** avec tous les démons chargés (`wazuh-agentd`, `wazuh-execd`, `wazuh-syscheckd`, etc.).

**Conclusion à ce stade :** le service tourne localement sur chaque agent, mais aucune communication effective n'a lieu avec le Manager — le problème se situe donc au niveau réseau ou configuration, pas au niveau du service lui-même.

### 2.1 Vérification du pare-feu sur le Manager (Rocky Linux)

```bash
sudo firewall-cmd --add-port=1514/tcp --permanent
sudo firewall-cmd --add-port=1515/tcp --permanent
sudo firewall-cmd --reload
```

```
Warning: ALREADY_ENABLED: 1514:tcp
success
Warning: ALREADY_ENABLED: 1515:tcp
success
success
```

**Résultat :** les ports 1514 (communication agent → manager) et 1515 (enrôlement) étaient **déjà ouverts**. Le pare-feu est donc écarté comme cause du problème.

**Conclusion :** le blocage vient très probablement du fichier de configuration des agents (`ossec.conf`), qui ne pointe pas vers la bonne adresse IP du Manager.

---

## 3. Correction de l'adresse du Manager via Ansible

Une nouvelle tâche a été ajoutée au playbook pour forcer l'adresse IP du Manager (`192.168.56.52`) dans le fichier `ossec.conf` de chaque agent, puis redémarrer le service :

```bash
ansible-playbook -i hosts.ini update_lab.yml
```

```
192.168.56.26  : ok=11  changed=3  failed=0
192.168.56.47  : ok=11  changed=2  failed=0
```

Nouvelle vérification côté Manager :

```bash
sudo /var/ossec/bin/agent_control -l
```

Le résultat restait identique — **aucun agent enregistré**. Le problème persistait malgré la correction de l'IP.

---

## 4. Découverte du vrai problème : incompatibilité de version Agent/Manager

L'analyse des journaux systemd a révélé l'erreur exacte :

```
ERROR: Agent version must be lower or equal to manager version (from manager)
```

### 4.1 Analyse

Le dépôt officiel Wazuh installait par défaut la **dernière version disponible de l'agent** (v4.14.x), alors que le Manager sur Rocky Linux tournait sur une version bien plus ancienne. Or Wazuh impose une règle stricte : **la version de l'agent ne peut jamais être supérieure à celle du Manager.**

### 4.2 Vérification de la version du Manager

```bash
sudo /var/ossec/bin/wazuh-control info
```

```
WAZUH_VERSION="v4.8.2"
WAZUH_REVISION="40819"
WAZUH_TYPE="server"
```

**Décision :** figer la version des agents installés par Ansible sur `4.8.2-1`, identique à celle du Manager, plutôt que de laisser `apt` installer la dernière version disponible.

### 4.3 Première correction (playbook v2)

```yaml
- name: Install Wazuh agent with Manager IP
  environment:
    WAZUH_MANAGER: "192.168.56.52"
  apt:
    name: wazuh-agent=4.8.2-1
    state: present
    allow_downgrades: yes
```

Cette modification seule n'a pas suffi : les machines ayant déjà installé la version 4.14.x, leur fichier `ossec.conf` existant contenait des paramètres propres aux versions récentes, non reconnus par la version 4.8.2.

---

## 5. Deuxième problème : configuration résiduelle incompatible (`journald`)

### 5.1 Symptôme

```bash
sudo journalctl -xeu wazuh-agent.service -n 20 --no-pager
```

```
ERROR: (1235): Invalid value for element 'log_format': journald.
ERROR: (1202): Configuration error at 'etc/ossec.conf'.
wazuh-logcollector: Configuration error. Exiting
wazuh-agent.service: Failed with result 'exit-code'.
```

### 5.2 Analyse

Lors de la première installation, la version 4.14.x avait généré un bloc `<log_format>journald</log_format>` dans `ossec.conf` — une option introduite dans les versions récentes de Wazuh. Après le downgrade vers la version 4.8.2, cette valeur n'était **plus reconnue**, ce qui empêchait le démarrage du service (`wazuh-logcollector` refusait de charger la configuration).

### 5.3 Tentative de correction ciblée (insuffisante)

```yaml
- name: Remove unsupported journald log_format from ossec.conf
  replace:
    path: /var/ossec/etc/ossec.conf
    regexp: '(?s)<localfile>\s*<log_format>journald</log_format>.*?</localfile>'
    replace: ''

- name: Set Wazuh Manager IP in ossec.conf
  lineinfile:
    path: /var/ossec/etc/ossec.conf
    regexp: '^\s*<address>.*</address>'
    line: '    <address>192.168.56.52</address>'

- name: Restart Wazuh agent service
  systemd:
    name: wazuh-agent
    state: restarted
    daemon_reload: yes
```

Cette approche corrective (patcher un fichier de config déjà pollué) restait fragile et n'a pas résolu le problème de façon fiable.

### 5.4 Solution définitive : purge complète avant réinstallation propre

Plutôt que de corriger un fichier de configuration hérité d'une mauvaise version, le playbook a été réécrit pour **purger entièrement** l'agent existant avant de réinstaller la bonne version depuis zéro :

```yaml
- name: Purge old broken Wazuh agent configurations
  apt:
    name: wazuh-agent
    state: absent
    purge: yes

- name: Install clean matching Wazuh agent version
  environment:
    WAZUH_MANAGER: "192.168.56.52"
  apt:
    name: wazuh-agent=4.8.2-1
    state: present

- name: Ensure Wazuh Manager IP is set in ossec.conf
  lineinfile:
    path: /var/ossec/etc/ossec.conf
    regexp: '^\s*<address>.*</address>'
    line: '    <address>192.168.56.52</address>'

- name: Enable and restart Wazuh agent service
  systemd:
    name: wazuh-agent
    enabled: yes
    state: restarted
    daemon_reload: yes
```

**Leçon technique retenue :** en cas de conflit de version Wazuh, purger (`purge: yes`) puis réinstaller la version cible est plus fiable que de corriger un fichier de configuration généré par une version différente — cela évite les résidus de configuration incompatibles.

### 5.5 Résultat de l'exécution finale du playbook

```bash
ansible-playbook -i hosts.ini update_lab.yml
```

```
192.168.56.26  : ok=11  changed=6  failed=0
192.168.56.47  : ok=11  changed=5  failed=0
```

Vérification côté Manager :

```bash
sudo /var/ossec/bin/agent_control -l
```

```
Wazuh agent_control. List of available agents:
   ID: 000, Name: localhost.localdomain (server), IP: 127.0.0.1, Active/Local
   ID: 001, Name: server, IP: any, Active
   ID: 002, Name: parrot, IP: any, Active
```

✅ **Résultat :** les deux agents (Ubuntu Server et Parrot OS) apparaissent désormais avec le statut **Active** dans le Manager.

---

## 6. Alignement manuel de l'agent Kali Linux

Kali Linux n'étant pas géré par Ansible (machine séparée sous VMware), la correction de version a été appliquée manuellement.

### 6.1 Vérification de la version installée

```bash
sudo /var/ossec/bin/wazuh-agentd -v
```

```
Wazuh v4.14.6 - Wazuh Inc.
```

Confirmation que Kali tournait avec la même version trop récente (4.14.6) que les autres agents avant correction.

### 6.2 Purge et réinstallation ciblée

```bash
sudo apt-get purge wazuh-agent -y
sudo WAZUH_MANAGER="192.168.56.52" apt-get install wazuh-agent=4.8.2-1 -y
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl restart wazuh-agent
```

Vérification finale :

```bash
sudo /var/ossec/bin/wazuh-agentd -v
```

```
Wazuh v4.8.2 - Wazuh Inc.
```

✅ Kali Linux est désormais aligné sur la version 4.8.2, compatible avec le Manager.

### 6.3 État du lab à ce stade

| Machine | Rôle | Version Wazuh | Statut |
|---|---|---|---|
| Rocky Linux | Manager | 4.8.2 | 🟢 Actif |
| Ubuntu Server | Agent | 4.8.2 | 🟢 Actif / Connecté |
| Parrot OS | Agent | 4.8.2 | 🟢 Actif / Connecté |
| Kali Linux | Agent | 4.8.2 | 🟢 Actif / Connecté |

<img width="1907" height="727" alt="image" src="https://github.com/user-attachments/assets/e94f025d-504d-4def-971e-41e446a1b70d" />


---

## 7. Mise en place d'un tunnel réseau pour le premier test d'attaque

### 7.1 Contexte et contrainte

Kali Linux (hôte d'attaque) est virtualisé sous **VMware**, tandis qu'Ubuntu Server (cible) est virtualisé sous **Hyper-V**. Bien que les deux VM affichent des adresses dans le même sous-réseau apparent (`192.168.56.x`), il s'agit en réalité de **deux réseaux virtuels totalement distincts et cloisonnés**, chacun géré par son propre hyperviseur — une simple coïncidence de plage d'adressage, sans connectivité réelle entre eux.

### 7.2 Première tentative : tunnel WireGuard direct

Un tunnel WireGuard a été configuré manuellement entre les deux machines.

**Sur Ubuntu Server :**
```bash
sudo wg genkey | sudo tee /etc/wireguard/privatekey | sudo wg pubkey | sudo tee /etc/wireguard/publickey
sudo nano /etc/wireguard/wg0.conf
sudo wg-quick up wg0
```
```
[#] ip -4 address add 10.8.0.1/24 dev wg0
[#] ip link set mtu 1420 up dev wg0
```

**Sur Kali Linux :**
```bash
sudo wg genkey | sudo tee /etc/wireguard/privatekey | sudo wg pubkey | sudo tee /etc/wireguard/publickey
sudo nano /etc/wireguard/wg0.conf
sudo wg-quick up wg0
```
```
[#] ip -4 address add 10.8.0.2/24 dev wg0
[#] ip link set mtu 1420 up dev wg0
```

**Résultat :** les interfaces WireGuard se sont montées correctement des deux côtés (`10.8.0.1` et `10.8.0.2`), mais la communication effective entre les deux hôtes restait bloquée.

### 7.3 Cause identifiée

Le problème vient de l'isolation totale des commutateurs virtuels :

- **VMware** crée son propre switch virtuel (`VMnet1`), attribuant à Kali l'adresse `192.168.56.129`.
- **Hyper-V** crée un switch virtuel indépendant, attribuant à Ubuntu l'adresse `192.168.56.26`.

Ces deux réseaux partagent la même plage d'adresses par coïncidence, mais **aucune passerelle ni pont ne les relie physiquement** — comme deux appartements avec la même numérotation de pièces, mais sans porte commune entre eux.

### 7.4 Décision : écarter le pont réseau manuel

La partie 1 avait déjà montré que les ponts réseau manuels (bridge Wi-Fi ↔ Hyper-V) fragilisaient sérieusement la stabilité du lab (pertes de connexion, conflits de routage). Pour ne pas reproduire cette instabilité, cette option a été délibérément écartée au profit d'une solution basée sur un **service VPN sur Internet**, qui ne modifie pas la configuration réseau locale des hyperviseurs.

### 7.5 Solution retenue : Tailscale (overlay VPN)

Contrairement à WireGuard configuré manuellement en pair-à-pair (qui nécessite un chemin réseau direct entre les deux hôtes), **Tailscale** utilise des serveurs relais (STUN/TURN) sur Internet pour établir la connexion, ce qui permet de relier deux machines situées dans des réseaux virtuels totalement isolés, sans toucher à la configuration des switches Hyper-V/VMware.

**Plan d'installation (préparé, à exécuter à la prochaine session) :**

1. Création d'un compte Tailscale gratuit (via Google ou GitHub).
2. Installation sur Ubuntu Server (cible) :
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up
   ```
3. Installation sur Kali Linux (attaquant) :
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up
   ```
4. Récupération de l'IP Tailscale d'Ubuntu (plage `100.x.y.z`) :
   ```bash
   tailscale ip -4
   ```
5. Test de connectivité depuis Kali :
   ```bash
   ping -c 4 <IP_TAILSCALE_UBUNTU>
   ```
6. Premier scénario d'attaque prévu — brute-force SSH via Hydra :
   ```bash
   hydra -l root -P /usr/share/wordlists/metasploit/common_passwords.txt <IP_TAILSCALE_UBUNTU> ssh
   ```
7. Observation des alertes générées en temps réel dans le **Wazuh Dashboard**.

---

## 8. Synthèse des problèmes et solutions (session du jour)

| Problème | Cause racine | Solution appliquée |
|---|---|---|
| Agents actifs localement mais absents du Manager | Adresse du Manager mal renseignée dans `ossec.conf` | Ajout d'une tâche Ansible pour fixer l'IP du Manager |
| Toujours aucun agent enregistré après correction IP | Version de l'agent (4.14.x) supérieure à celle du Manager (4.8.2) — incompatibilité protocolaire | Fixer la version installée à `wazuh-agent=4.8.2-1` |
| Service en échec après downgrade (`log_format: journald`) | Résidu de configuration généré par la version précédente (4.14.x), incompatible avec 4.8.2 | Purge complète (`purge: yes`) puis réinstallation propre de la version cible |
| Kali Linux non couvert par Ansible | Machine hors inventaire (VMware, hors lab Hyper-V) | Purge + réinstallation manuelle de la version 4.8.2 |
| Kali (VMware) et Ubuntu (Hyper-V) dans la même plage IP mais non connectés | Deux switches virtuels totalement isolés (VMnet1 / Hyper-V vSwitch) malgré un adressage identique | Abandon du pont réseau manuel (instable) au profit d'un VPN overlay (Tailscale) |

---

## 9. État actuel du lab

- ✅ Wazuh Manager (Rocky Linux, v4.8.2) opérationnel.
- ✅ 3 agents Wazuh enregistrés et actifs : Ubuntu Server, Parrot OS, Kali Linux (tous alignés en v4.8.2).
- ✅ Playbook Ansible fiabilisé : gestion du verrou apt, purge propre avant réinstallation, configuration de l'IP du Manager, redémarrage automatique du service.
- 🔜 Tunnel réseau Kali ↔ Ubuntu à établir via Tailscale (WireGuard manuel abandonné en raison de l'isolation des switches virtuels).

## 10. Prochaines étapes

1. Déployer Tailscale sur Ubuntu Server et Kali Linux.
2. Valider la connectivité inter-hyperviseurs via l'IP overlay Tailscale (`100.x.y.z`).
3. Lancer un premier scénario d'attaque (brute-force SSH avec Hydra) depuis Kali vers Ubuntu.
4. Observer et documenter la détection des alertes correspondantes dans le Wazuh Dashboard (règles déclenchées, niveau de sévérité, temps de détection).
5. Étendre les scénarios de test à d'autres techniques MITRE ATT&CK (scan de ports, exploitation de service, persistance).

---

*Rapport rédigé dans le cadre du projet personnel de Home Lab SOC, destiné au portfolio GitHub — suite directe du Rapport 1.*
