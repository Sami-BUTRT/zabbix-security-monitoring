# Zabbix Security Monitoring

Déployer, configurer et surveiller un réseau avec **Zabbix 7.0** sur des instances **Ubuntu Cloud** hébergées sur **DigitalOcean**.

---

## Table des matières

1. [Présentation du projet](#présentation-du-projet)
2. [Architecture en étoile](#architecture-en-étoile)
3. [Configuration de l'agent Zabbix](#configuration-de-lagent-zabbix)
4. [Alertes de sécurité — Charge CPU](#alertes-de-sécurité--charge-cpu)
5. [Prérequis](#prérequis)

---

## Présentation du projet

Ce projet met en place une solution de **supervision réseau** centralisée basée sur Zabbix 7.0. L'infrastructure repose sur des Droplets Ubuntu (22.04 LTS) déployées sur DigitalOcean et communique via une architecture en étoile (*hub-and-spoke*).

Objectifs principaux :
- Surveiller la disponibilité et les performances des hôtes distants.
- Détecter les anomalies de charge CPU et déclencher des alertes de sécurité.
- Centraliser les logs et métriques dans un tableau de bord unique.

---

## Architecture en étoile

```
                      ┌─────────────────────┐
                      │   Zabbix Server      │
                      │  (Droplet Ubuntu)    │
                      │  IP : 10.0.0.1       │
                      └─────────┬───────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
   ┌──────────┴──────┐ ┌────────┴───────┐ ┌──────┴──────────┐
   │  Client 1        │ │  Client 2      │ │  Client N       │
   │  (Droplet Ubuntu)│ │ (Droplet Ubuntu│ │ (Droplet Ubuntu)│
   │  IP : 10.0.0.2   │ │ IP : 10.0.0.3  │ │ IP : 10.0.0.N   │
   └──────────────────┘ └────────────────┘ └─────────────────┘
```

### Rôle du Serveur

| Composant | Description |
|-----------|-------------|
| **Zabbix Server** | Collecte, stocke et analyse les données de supervision |
| **Zabbix Web** | Interface graphique (Apache2 + PHP 8.x) |
| **Base de données** | MariaDB / PostgreSQL (stockage des métriques et alertes) |

### Rôle des Clients (Agents)

| Composant | Description |
|-----------|-------------|
| **Zabbix Agent 2** | Collecte locale des métriques (CPU, RAM, disque, réseau) |
| **Communication** | Port TCP **10050** (agent passif) ou **10051** (agent actif) |
| **Sécurité** | Connexions autorisées uniquement depuis l'IP du serveur |

> **Note DigitalOcean :** Utiliser le réseau VPC privé entre les Droplets pour éviter d'exposer le port 10050 sur l'interface publique.

---

## Configuration de l'agent Zabbix

### Installation sur un client Ubuntu

```bash
# Télécharger le dépôt officiel Zabbix 7.0
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest+ubuntu22.04_all.deb
sudo dpkg -i zabbix-release_latest+ubuntu22.04_all.deb
sudo apt update

# Installer l'agent Zabbix 2
sudo apt install -y zabbix-agent2
```

### Fichier de configuration `/etc/zabbix/zabbix_agent2.conf`

```ini
# Adresse du serveur Zabbix (IP privée DigitalOcean)
Server=10.0.0.1
ServerActive=10.0.0.1

# Nom d'hôte unique — doit correspondre à celui déclaré dans Zabbix Web
Hostname=client-01

# Port d'écoute de l'agent
ListenPort=10050

# Logs
LogFile=/var/log/zabbix/zabbix_agent2.log
LogFileSize=10

# Délai de rafraîchissement des métriques actives (secondes)
RefreshActiveChecks=120
```

### Démarrer et activer l'agent

```bash
sudo systemctl enable --now zabbix-agent2
sudo systemctl status zabbix-agent2
```

### Règle de pare-feu UFW (sur le client)

```bash
# Autoriser uniquement le serveur Zabbix à accéder au port de l'agent
sudo ufw allow from 10.0.0.1 to any port 10050 proto tcp
sudo ufw reload
```

### Ajouter le client dans Zabbix Web

1. Aller dans **Configuration → Hôtes → Créer un hôte**.
2. Renseigner le **Nom d'hôte** (doit correspondre à `Hostname` dans le fichier de conf).
3. Ajouter l'**Interface Agent** avec l'IP privée du client et le port `10050`.
4. Associer le template **`Linux by Zabbix agent`**.
5. Sauvegarder et vérifier que le statut passe à **ZBX** (vert).

---

## Alertes de sécurité — Charge CPU

### Principe

Une alerte est déclenchée lorsque la charge CPU moyenne dépasse un seuil critique pendant une durée définie, ce qui peut indiquer une attaque par déni de service (DoS), un processus malveillant ou une surcharge anormale.

### Créer un déclencheur (*Trigger*) CPU

1. Aller dans **Configuration → Hôtes** → sélectionner l'hôte cible.
2. Cliquer sur **Déclencheurs → Créer un déclencheur**.
3. Renseigner les champs suivants :

| Champ | Valeur |
|-------|--------|
| **Nom** | `Charge CPU élevée — Alerte sécurité` |
| **Sévérité** | `Haute` |
| **Expression** | `avg(/client-01/system.cpu.util[,user],5m)>85` |
| **Description** | Charge CPU utilisateur > 85 % sur 5 minutes — intervention requise |

> Adapter `client-01` au nom de l'hôte réel et le seuil `85` selon la politique de sécurité.

### Expression Zabbix expliquée

```
avg(/client-01/system.cpu.util[,user],5m) > 85
│    │         │                    │      │
│    │         └─ Métrique : % CPU  │      └─ Seuil critique
│    └─ Nom d'hôte                  └─ Fenêtre de 5 minutes
└─ Fonction : moyenne sur la période
```

### Configurer une action d'alerte (*Action*)

1. Aller dans **Configuration → Actions → Déclencheurs d'actions → Créer une action**.
2. Nommer l'action : `Alerte CPU critique`.
3. **Condition :** Déclencheur = `Charge CPU élevée — Alerte sécurité`.
4. **Opération :** Envoyer un message à l'utilisateur/groupe concerné.
5. **Média :** Configurer un canal (Email, Telegram, PagerDuty…) dans **Administration → Alertes → Types de médias**.

### Exemple de message d'alerte

```
Sujet : [ALERTE] Charge CPU critique sur {HOST.NAME}

Hôte  : {HOST.NAME}
IP    : {HOST.IP}
Heure : {EVENT.DATE} {EVENT.TIME}
Valeur: {ITEM.VALUE}

Action requise : Vérifier les processus en cours d'exécution.
Commande : top -b -n 1 | head -20
```

---

## Prérequis

| Élément | Version / Détail |
|---------|-----------------|
| Zabbix Server & Agent | 7.0 (LTS) |
| Système d'exploitation | Ubuntu 22.04 LTS |
| Fournisseur Cloud | DigitalOcean (Droplets) |
| Base de données | MariaDB 10.11+ ou PostgreSQL 15+ |
| PHP | 8.x |
| Réseau | VPC DigitalOcean (communication inter-Droplets) |

---

*Projet réalisé dans le cadre d'un atelier de supervision réseau et sécurité.*
