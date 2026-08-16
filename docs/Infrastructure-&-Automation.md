# 🛡️ Laboratoire personnel : Infrastructure & Automation

Bienvenue dans le dépôt officiel de mon laboratoire de cybersécurité. Ce projet documente la mise en place d'une infrastructure automatisée dédiée à l'analyse réseau, au monitoring et à la défense (Blue Team).

---

## 🏗️ 1. Architecture du Laboratoire (Physical Components)

Le laboratoire est structuré autour d'un réseau local robuste et de nœuds de virtualisation.

| Composant | Adresse IP | Rôle |
|---|---|---|
| **Giada Mini PC** | `10.0.0.100:8006` | Hyperviseur (Proxmox VE) |
| **Linksys Velop** | `10.0.0.1` | Routeur Principal (OpenWrt) |
| **D-Link Router** | `10.0.0.10` | Routeur Secondaire (ImmortalWrt) |
| **Panasonic PC** | `10.0.0.50` | Nœud Géré (Managed Node) |
| **WSL Ubuntu** | `172.30.137.208` | Contrôleur Ansible (Control Node) |
| **Kali VM** | `192.168.56.128` | Serveur de Laboratoire (Lab Server) |

> **Note :** pour les IPs dynamiques, une migration vers une réservation via adresse MAC est recommandée.

---

## 🔐 2. Couche Réseau Virtuelle (VPN Layer)

Isolation du trafic via un tunnel **WireGuard** :

- **Sous-réseau** : `10.10.0.0/24`
- **Kali (Serveur)** : `10.10.0.1`
- **WSL (Client)** : `10.10.0.2`

```bash
sudo wg-quick up wg0
ip a show wg0
```

---

## ⚙️ 3. Gestion des Inventaires (Ansible)

- **Global Inventory** (`hosts`) — topologie physique (`10.0.0.x`)
- **VPN Inventory** (`hosts_panasonic_vpn`) — nœuds via tunnel (`10.10.0.x`)

**Test de connectivité :**

```bash
ansible lab_servers -i hosts_panasonic_vpn -m ping
```

**Résultat :**

```
SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

## 🛡️ 4. Sécurisation (SSH)

Automatisation sans mot de passe :

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
ssh-copy-id kali@10.10.0.1
```

---

## 📈 5. Intégration Wazuh

Le système **Wazuh** est intégré via le VPN (`10.10.0.1`) pour une intégrité optimale des logs.

---

## 💼 Compétences & Profil

- **SecOps Engineer** — surveillance et gestion d'actifs
- **TDIR** — analyse des menaces en temps réel (Threat Detection, Investigation & Response)
- **Blue Team** — isolation réseau et automatisation

---

## 📝 Rapports complémentaires

- [Consulter le rapport d'évolution détaillé](docs/mise%20a%20jour%20labo.md)

---

Maintenu par **Youssef Ghraba**
