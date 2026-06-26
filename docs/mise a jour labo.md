# Laboratoire de Cybersécurité : Rapport d'Évolution et Ingénierie

## Résumé Exécutif
Ce projet documente l'évolution architecturale d'un laboratoire de cybersécurité. Ce rapport retrace le parcours technique complet, passant d'un prototype expérimental (WSL/VMware) à une infrastructure robuste et centralisée (Proxmox/OPNsense), conçue pour simuler des environnements de type SOC (Security Operations Center).

---

## 1. Topologie et Architecture
Le laboratoire est structuré pour garantir l'isolation et la performance nécessaire aux tests d'intrusion et à la détection des menaces.

| Composant | Rôle | Adresse IP |
| :--- | :--- | :--- |
| **Giada (Proxmox)** | Hyperviseur Principal | 10.0.0.100 |
| **Linksys Velop** | Passerelle Internet | 10.0.0.1 |
| **D-Link (ImmortalWrt)** | Sandbox (Isolation) | 10.0.0.10 |
| **Panasonic (Hyper-V)** | Station de travail & Contrôleur | 10.0.0.50 |

---

## 2. Parcours Technique : Historique des Manipulations

### Phase I : Le Prototype (WSL + VMware)
* **Objectif :** Établir une communication entre Wazuh Manager (WSL) et Kali Linux (VM) via `WireGuard`.
* **Actions réalisées :**
    * Installation de `WireGuard` sur WSL et Kali.
    * Génération de clés cryptographiques (`wg genkey | tee privatekey | wg pubkey > publickey`).
    * Création des interfaces VPN (`wg-quick up wg0`) et tests de connectivité (`ping -c 4 10.10.0.1`).
* **Défis :** Lors de l'inversion des adresses IP, le "Handshake" a échoué de manière répétée. La virtualisation imbriquée créait des conflits de routage NAT irrésolubles manuellement.

### Phase II : Tentatives de Correction et Stabilisation
* **Actions de dépannage (Root Cause Analysis) :**
    * Suppression des anciennes interfaces (`ip link delete dev wg0`) et réinitialisation des proxys (`netsh interface portproxy reset`).
    * Création de nouvelles paires de clés pour WSL, Kali et Windows.
    * Configuration du `.wslconfig` en mode `NAT` pour tenter de stabiliser la pile réseau.
* **Leçon apprise :** Une surveillance de sécurité fiable nécessite une infrastructure réseau pérenne et non des tunnels logiciels temporaires gérés manuellement sur des hôtes Windows.

---

## 3. Phase III : Pivot Stratégique et Renforcement
* **Action :** Migration vers un environnement `Hyper-V` dédié et déploiement d'un routeur `OPNsense` sur `Giada`.
* **Changements majeurs :**
    * Centralisation du routage via `OPNsense` avec des baux statiques (DHCP Static Mapping), éliminant les conflits d'IP.
    * Conteneurisation des services (Wazuh Manager) sous `LXC` (Proxmox) pour optimiser les ressources.
* **Résultat :** Élimination totale des problèmes de "Handshake". Le réseau est désormais stable, scalable et professionnel.

---

## 4. Automatisation et Déploiement (Infrastructure as Code)
Afin d'assurer la reproductibilité, l'automatisation est gérée par `Ansible`.

### Flux de configuration :
1. **Gestion des inventaires :** Distinction entre l'inventaire physique (`hosts`) et l'inventaire VPN (`hosts_panasonic_vpn`).
2. **Provisionnement automatisé :**
   ```yaml
   - name: Configurer l'agent Wazuh via VPN
     hosts: all
     become: yes
     tasks:
       - name: Définir l'adresse du Manager
         replace:
           path: /var/ossec/etc/ossec.conf
           regexp: '<address>.*</address>'
           replace: '<address>10.10.0.1</address>'
         notify: restart wazuh-agent

         Sécurité SSH : Déploiement automatisé de clés RSA-4096 pour une communication sécurisée sans mot de passe entre le contrôleur (Panasonic) et les nœuds (Kali/Giada).

## 5. Vision Future et Compétences
Segmentation Avancée : Implémentation de VLANs sur OPNsense pour isoler davantage le trafic des tests d'intrusion.

Monitoring : Déploiement d'un conteneur LXC dédié au Wazuh Dashboard pour séparer l'indexation de la gestion.

Compétences Démontrées :

Ingénierie SecOps : Gestion continue et cycle de vie des actifs de sécurité.

Architecture Réseau : Conception de topologies robustes avec OPNsense et Proxmox.

Automatisation (IaC) : Maîtrise d'Ansible pour le déploiement sécurisé à grande échelle.
