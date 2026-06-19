# Rapport Technique : Infrastructure et Automatisation du Laboratoire

## 1. Architecture du Laboratoire (Physical Components)
Le laboratoire est structuré autour d'un réseau local robuste et de nœuds de virtualisation.

| Composant | Adresse IP | Rôle |
| :--- | :--- | :--- |
| **Giada Mini PC** | 10.0.0.100:8006 | Hyperviseur (Proxmox VE) |
| **Linksys Velop** | 10.0.0.1 | Routeur Principal (OpenWrt) |
| **D-Link Router** | 10.0.0.10 | Routeur Secondaire (ImmortalWrt) |
| **Panasonic PC** | 10.0.0.50 | Nœud Géré (Managed Node) |
| **WSL Ubuntu** | 172.30.137.208 | Contrôleur Ansible (Control Node) |
| **Kali VM** | 192.168.56.128 | Serveur de Laboratoire (Lab Server) |

> **Note sur la persistance des IPs :** Pour les IPs dynamiques (Kali et WSL), il est recommandé de migrer vers une réservation via adresse MAC sur le serveur DHCP (OpenWrt) ou via OPNsense pour une gestion centralisée et professionnelle.

## 2. Couche Réseau Virtuelle (VPN Layer)
Pour sécuriser les communications, un tunnel **WireGuard** a été déployé, isolant le trafic du laboratoire du reste du réseau domestique.

* **Nom du VPN :** `vpn`
* **Sous-réseau :** `10.10.0.0/24`
* **Kali (Serveur) :** `10.10.0.1`
* **WSL (Client) :** `10.10.0.2`

### Commandes de gestion VPN :
```bash
# Activation du tunnel
sudo wg-quick up wg0
# Vérification de l'interface
ip a show wg0 
**3. Gestion des Inventaires (Ansible Inventory)
Afin de dissocier les environnements de test, deux inventaires distincts sont utilisés :

Global Inventory (hosts) : Gère la topologie physique (10.0.0.x).

VPN Inventory (hosts_panasonic_vpn) : Gère les nœuds via le tunnel (10.10.0.x).

Résultat de vérification (Ping Test) :
Bash
ansible lab_servers -i hosts_panasonic_vpn -m ping
Résultat :

JSON
wsl_client | SUCCESS => { "changed": false, "ping": "pong" }
kali_server | SUCCESS => { "changed": false, "ping": "pong" }
4. Sécurisation et Automatisation (SSH & Security)
Le déploiement des services SSH et la configuration des clés garantissent une automatisation sans mot de passe.

Installation et configuration SSH :
Bash
sudo apt update && sudo apt install openssh-server -y
sudo systemctl enable --now ssh
Authentification par clés :
Bash
# Génération de la paire de clés
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""

# Déploiement des clés publiques
ssh-copy-id kali@10.10.0.1
ssh-copy-id jow@10.10.0.2
5. Intégration Wazuh
Le système Wazuh est désormais intégré dans le flux sécurisé du VPN. Pour optimiser l'intégrité des logs, les agents communiquent exclusivement via l'interface 10.10.0.1.

6. Conclusion et Évolutivité
Cette infrastructure permet désormais :

Isolation totale du trafic sensible.

Automatisation complète via les Playbooks Ansible.

Évolutivité facilitée par l'ajout dynamique d'hôtes dans le fichier hosts_panasonic_vpn.

7. Compétences et Profil Professionnel
Ce laboratoire a été conçu pour démontrer des compétences pratiques alignées avec les exigences des rôles suivants :

Security Operations Engineer (SecOps) : Maîtrise de la surveillance continue et de la gestion des actifs.

Threat Detection & Incident Response (TDIR) : Intégration de solutions comme Wazuh pour l'analyse des menaces en temps réel.

Blue Team Engineer : Isolation réseau, sécurisation des accès SSH et automatisation de la défense.

Ce projet témoigne de mon expérience dans la mise en place d'infrastructures sécurisées, l'automatisation via Ansible et la gestion des flux de logs, compétences clés pour un ingénieur en cybersécurité